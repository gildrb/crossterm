# Patches

This branch is crossterm 0.28.1 (upstream commit `492b764`, the source of the
crates.io release) with the changes below, one commit each, in this order.
Eukhe (https://github.com/gildrb/eukhe) depends on it through
`[patch.crates-io]`, pinned by commit. Everything else is upstream crossterm,
under its MIT license (`LICENSE`).

The crate's tests: `cargo test --locked --features event-stream --lib`.

## 1. The keyboard enhancement query answers within 250ms

Problem: `supports_keyboard_enhancement` waited up to 2000ms for the kitty
flags answer while holding the process-wide event-reader lock. On a terminal
that never answers the flags query (tmux and screen answer only DA1; a bare
pty answers nothing), input typed meanwhile reached the application two
seconds late.

Change: the answer window is 250ms, and the check does not hold the reader
lock through it, so another reader delivers input during the window. A reply
read by that reader stays in the shared queue for the check to find. Entry 8
gives the lock handling its final form.

Pinned by: Eukhe's `kitty_verdict_time_e2e` (a silent pty concludes at
the 250ms deadline; a DA1 answer concludes the check at its arrival).

## 2. `poll_opt`, `waker` and `Waker` are public

Problem: a thread reading events that must also stop on request could only
poll with a timeout and wake on every tick; the event source's waker was
crate-private.

Change, additive only: `event::poll_opt(Option<Duration>)` (`None` parks until
input or a wake; a wake returns `Ok(false)` like a timeout), `event::waker()`
(`None` when the event source failed to open, e.g. no controlling tty),
public `Waker` and `Waker::wake`, and `InternalEventReader::try_waker`, which
returns `None` instead of panicking.

Pinned by: Eukhe's input reader (`crates/eukhe-tui/src/input.rs`) and its
input tests; no test in this crate.

## 3. `supports_keyboard_enhancement_checked_raw`

Problem: `supports_keyboard_enhancement` enables raw mode itself when it finds
raw mode off. A caller that checked raw mode first can race a concurrent
hand-back of the terminal to the shell (a suspend), and the check then
enables raw mode on the shell's terminal.

Change: `terminal::supports_keyboard_enhancement_checked_raw` runs only the
query, for callers that hold raw mode themselves, and never changes the
terminal mode.

Pinned by: Eukhe's `kitty_exit_routes_e2e` and `suspend_signal_e2e`; no test
in this crate.

## 4. No compiler warnings on current Rust

Problem: Rust 1.80 and later warn on unknown cfg names: the `compile_error!`
guards on `cfg(winapi)` and `cfg(crossterm_winapi)` never fired and warned on
every build. `InternalEventFilter` is used only by tests (`dead_code`), and
one closure body had redundant parentheses (`unused_parens`).

Change: the two dead guards are removed, `InternalEventFilter` is built for
tests only, and the parentheses are gone.

Pinned by: `cargo build --features event-stream` printing no warnings.

## 5. The tty drains on every readiness edge

Problem: mio reports the tty edge-triggered, but the source returned after one
1024-byte read. Bytes still queued raised no new edge, so text written in one
write (a paste without bracketed paste mode) stopped after 1024 bytes until
the next keypress, and its trailing Enter was lost.

Change: the source reads until `FIONREAD` reports nothing pending (the first
read is unconditional so a hangup surfaces). The parser's `more` flag means
"bytes still queued", so a sequence split across reads stays open and a full
read ending in a lone ESC is not cut. Read errors other than EINTR and
EAGAIN are returned.

Pinned by: Eukhe's `anthropic_remote_login_pty_e2e`
`a_typed_long_redirect_in_one_write_logs_in`.

## 6. modifyOtherKeys reports of modified control keys

Problem: with the kitty protocol inactive, Ghostty reports keys such as
Shift+Enter, Ctrl+Tab and Alt+Backspace as `CSI 27 ; <modifiers> ;
<codepoint> ~`. The parser rejected that form and the keys were lost.

Change: the form decodes to a key event with its modifiers; a missing or
extra field or an invalid codepoint is still an error.

Pinned by: `event::sys::unix::parse` `test_parse_modify_other_keys`.

## 7. `KeyEventState::BARE_CSI_U`

Problem: a character in the bare kitty form `CSI <codepoint> u` and the same
character as raw text parsed to identical events. Some kitty terminals report
one keypress in both forms; an application dropping the raw twin could not
tell it from a typed repeat and dropped repeated characters ("300" became
"30").

Change: a `Char` press decoded from the bare form (digits only: no alternate
keys, modifier or event-type field) carries `KeyEventState::BARE_CSI_U`. A
form with a modifier field, even `;1`, does not.

Pinned by: `event::sys::unix::parse` `test_parse_basic_csi_u_encoded_key_code`;
Eukhe's `raw_repeated_characters_in_one_kitty_burst_all_arrive`.

## 8. The reader lock reaches the keyboard check fairly

Problem: under load the check missed answers already in the queue. It waited
for the reader lock 10ms at a time, and a reader polling in a loop re-took
the lock at once after each release, so the check lost every round of its
window.

Change: the check waits for the lock for the whole remaining window and reads
the answer under that guard. It never waits on the tty while holding the
lock: each look is a zero-timeout poll, then it releases with `unlock_fair`
and sleeps 1ms. `poll_internal` and `read_internal` also release with
`unlock_fair`, so a waiting check gets the lock at the next release.

Pinned by: `terminal::sys::unix` tests
`a_queued_answer_reaches_the_probe_past_a_reader_that_keeps_the_lock` and
`a_probe_on_a_silent_tty_never_waits_on_it_with_the_reader_locked` (both fail
on the version before this change); Eukhe's
`terminal_state_differential_e2e` and `kitty_early_typing_e2e`.

## 9. The keyboard check looks once more at its deadline

Problem: on a loaded box the check concluded "no answer" while the
terminal's answer had long been read into the shared queue: its thread woke
past its 250ms deadline and gave up without looking, or the app reader that
had queued the answer was itself off the CPU inside its bounded poll with
the reader lock held, and the check's lock wait ended with the window. The
session then ran without the keyboard protocol.

Change: the window closes with a look. Past the deadline the check waits
for the reader lock without a time limit and looks once before it reports
the timeout. That wait ends whenever no reader holds the lock while it waits
on the tty with no timeout (a blocking `read` with no `poll` first): every
other holder releases at the end of one bounded poll. Eukhe polls before
each read (its input loop, the selectors and the drains poll for at most
50ms).

Pinned by: `terminal::sys::unix` tests
`a_probe_waking_past_its_deadline_still_finds_its_queued_answer` and
`a_reader_holding_the_lock_past_the_window_still_hands_over_the_answer`
(both fail on the version before this change); Eukhe's
`terminal_state_differential_e2e` under CPU starvation
(`the_config_selector_esc_close_restores_every_mode` failed 27 of 40 runs at
nice 19 beside 48 busy loops).

## Updating to a newer upstream release

Rebase these commits onto the upstream commit of the new release (its
`.cargo_vcs_info.json` on crates.io names it), resolve conflicts, run the
tests, push a new branch `eukhe-<version>` and tag `eukhe-<version>-1`
(`-2`, `-3` for later revisions of the same base), then bump the `rev` in
Eukhe's `Cargo.toml` and its `Cargo.lock`.
