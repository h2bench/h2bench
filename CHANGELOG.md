# Changelog

Release notes for the agent, newest first. Binaries and their checksums are on the
[Releases page](https://github.com/h2bench/h2bench/releases).

## 2.2.11

Dashboard and reporting changes, and a working global hotkey.

- **Header.** The Dashboard greeting was replaced by a single line carrying what a result is
  judged against: GPU, CPU, memory, driver version, and one state word. The operating system and
  display line that briefly occupied the slot was removed again; the machine plate to the right
  of the same banner already showed both.
- **Duplicate status removed.** The "API connected" notice no longer appears at start-up. The
  header chips and the sidebar already carried that state, and a banner that repeats a permanent
  status is one a reader learns to ignore. Notices for things that did just happen now close
  themselves after four seconds; warnings and errors stay until dismissed.
- **Publish choice is now a real control.** The Benchmark page offered two paragraphs describing a
  setting that did not exist: both Start buttons had recording and server verification hardcoded
  on. It is now an explicit choice between *Local only* and *Record & publish to H2Bench*, with a
  Video proof badge, and both Start buttons follow it.
- **Chart.** An FPS / Frametime switch replaces the legend, and a summary of AVG, 1% LOW and
  0.1% LOW sits directly above the graph. The summary changes unit with the switch, so it cannot
  be read as frames per second against a millisecond curve.
- **Global hotkey.** `Ctrl + Shift + O` shows and hides the live overlay from any window. With the
  monitor off it starts the card for the running game; with it on it hides and shows the card
  without stopping the measurement. The Settings section previously listed three combinations as
  static text: none were registered, one described a behaviour the live overlay deliberately does
  not have, and one described a feature that does not exist. The two that described nothing real
  are gone.
- **Live overlay button** is now styled as a peer of Start Bench rather than as a secondary
  control, and is named FPS Monitor.
- Start Bench is the name on all three start buttons: Dashboard, Benchmark and the game card.

## Earlier versions

Notes for 2.0.1 through 2.2.1 exist in the project's internal release documents and were written
for a different audience; they are not reproduced here yet. Versions 2.2.2 through 2.2.10 were
distributed without release notes being recorded.

What is known about the 2.2.x line: the presentation layer was migrated to WinUI 3, the agent was
rebuilt as a self-contained single-file application, a frame-pacing graph was added to the
overlay, and a free live-overlay mode was introduced that records nothing.
