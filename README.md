# Tattoy: Eye-candy for your terminal

Currently running with:

```
TERM=xterm-256color SHELL=zsh RUST_BACKTRACE=1 RUST_LOG="none,shadow_terminal=trace,tattoy=trace" cargo run -- --use smokey_cursor
```

Testing with:

```
cargo build --package tattoy && cargo test -- --nocapture
```

In CI I use `cargo nextest run --retries 1` because some of the e2e tests are flakey.

Generate docs with:
`cargo doc --no-deps --document-private-items --open`

Logs go to: `./tattoy.log`

## Usage
Scroll wheel scrolls the scrollback. `<ESC>` exits scrolling (TODO: enable other keybindings for scrolling).


## TODO
* [x] Background colour of " " (space) isn't passed through.
* [x] Bold doesn't get passed through properly, run `htop` to see.
* [x] Resizing isn't detected.
* [x] Cursor isn't transparent.
* [x] Send surface updates to state only, then protocol sends small signal not big update.
* [x] Look into performance, especially scrolling in nvim.
* [x] `CTRL-D` doesn't fully return to terminal, needs extra `CTRL-C`.
* [x] Resizing is broken.
* [x] Look at projects like Ratatui to see how to do integration tests.
* [x] Use `tokio::select!` in Loader
* [x] Explore rendering a frame even if any of the surfaces aren't the right size, in order to not prevent updates from other surfaces.
* [x] Explore returning errors in tasks/threads, joining them and acting on them. Instead of sending the error to shared state.
* [x] Centralise place where app exits and outputs backtrace and messages etc.
* [x] Implement scrollback/history.
* [x] Palette detection.
* [x] Ask for consent before taking screen shot.
* [x] Support manually providing a screenshot of the palette.
* [x] Double width characters aren't passed through, eg "🦀".
* [x] `tmux` mouse events cause runaway behaviour in `htop`.
* [x] Mouse events not being received in alternate screen.
* [ ] Extra minimap features: https://www.youtube.com/shorts/t5vXCNIBVYw
* [ ] Pasting is buggy
* [ ] Make all config optional
* [ ] How should smokey_cursor particles respond to resizing?
* [ ] Detect alternate screen so to hide cursor
* [ ] Up and down aren't detected in `less` or `htop`.
* [ ] User-friendly error output for known errors
* [ ] Bug: `atuin` can't get the cursor position. Maybe I need to intercept '\e[6n'?
* [ ] Don't log to file by default
* [ ] Tattoy-specific keybinding to toggle all tattoys on and off.
* [ ] Allow scrolling with keys.
* [ ] Chafa sometimes emits an unknown char, but for the same image it always work fine outside of Tattoy.
* [ ] Chafa on large terminals often causes Tattoy to completely freeze.
* [ ] Large terminal screens sometimes cause scrollback to stop rendering.
* [ ] More profiling. I tried https://github.com/mstange/samply and https://github.com/flamegraph-rs/flamegrap but they had some obscure errors which I assumed were from my CPU architecture, Asahi M1 etc.
* [ ] Show a nice notification when the user tries to enable a tattoy that requires a parsed palette, but they haven't parsed their palette yet. And of course also don't try to even render such palettes.
* [ ] Make a website showcasing and providing docs.
* [ ] Make flakey tests reliable.
* [ ] How to test the minimap?
* [ ] When scrolling the Shadow Terminal itself should refuse all input. Currently Tattoy is the one surpressing input.
* [ ] Doesn't work on Nushell. Just freezes.
* [ ] Pixel top uses raw black not default colour.

## Brand and Lore
* The terminal has a reputation of being dark and esoteric, which is good. But can we bring some lightness and folly to it, whilst retaining the Seriousness™️?
* What _is_ a terminal? Where did it come from? What's it made from? Does it have physicality?

## Design

### Terminals/Surfaces
There are quite a few terminals, PTYs, shadow PTYs, surfaces, etc, that are all terminal-like in some way, but do different things.

* __The user's actual real terminal__ We don't really have control of this. Or rather, Tattoy as an application merely is a kind of magic trick that reflects the real terminal whilst sprinkling eye-candy onto it. The goal of Tattoy is that you should _always_ be able to recover your original untouched terminal.
* __The PTY (pseudo TTY) of the "original" terminal process__ To achieve the magic trick of Tattoy we manage a "shadow" subprocess of the user's real terminal. It is managed completely in memory and is rendered headlessly by yet another "terminal" (see shadow TTY). The PTY code itself is provided by the [portable_pty](https://docs.rs/portable-pty/latest/portable_pty/) crate from the [Wezterm project](https://github.com/wez/wezterm) ❤️.
* __The shadow PTY of the "original" terminal screen__ This is just a headless rendering of the underlying shadow PTY. It is a virtual terminal. It is a purely in-memory representation of the PTY and hence of the user's original terminal. This is done with a [wezterm_term::Terminal](https://github.com/wez/wezterm/blob/main/term/README.md).
* __The Tattoy magic surface__ A surface here refers to a [termwiz::surface::Surface](https://github.com/wez/wezterm/tree/main/termwiz). It represents a terminal screen, but is not an actual real terminal, it's merely a convenient visual representation. This is where we can create all the magical Tattoy eye-candy. Although it does not intefere with the shadow TTY, it can be informed by it. Hence why you can create Tattoys that seem to interact with the real terminal. In the end, this Tattoy surface is composited with the contents of the shadow PTY.
* __The shadow PTY surface__ This is merely a copy of the current visual status of the shadow TTY. We don't use the actual shadow TTY as the source because it's possible that this data is queried frequently by various Tattoys. Querying the a static visual representation is more efficient than querying a TTY, even if it exists only in memory.
* __The final composite surface__ This is the final composited surface of the both the underlying shadow PTY and all the active Tattoys. A diff of this with the user's current real terminal is then used to do the final update.
