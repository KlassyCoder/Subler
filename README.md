# README #

### What is this repository for? ###

Subler is an Mac OS X app created to mux and tag mp4 files. The main features includes:

* Creation of tx3g subtitles tracks, compatible with all Apple's devices (iPod, AppleTV, iPhone, QuickTime?).
* Mux video, audio, chapters, subtitles and closed captions tracks from mov, mp4 and mkv.
* Raw formats: H.264 Elementary streams (.h264, .264), AAC (.aac), AC3 (.ac3), Scenarist (.scc), VobSub? (.idx).
* metadata editing and TMDb, TVDB and iTunes Store support.

### Build and run

Clone the repository and include all submodules
```
git clone --recurse-submodules https://github.com/SublerApp/Subler.git
```
If you already cloned without submodules and need to add the submodules manually, `cd` into the `./Subler` directory and clone the dependency submodules with 
```
git submodule update --init --recursive
```
Open `Subler.xcodeproj` in XCode

Build and run the project by selecting the 'Subler' scheme (`Product` -> `Scheme` -> `Subler`) and clicking the 'Run' button in Xcode’s toolbar.

### Fork changes (KlassyCoder)

This fork is signed and notarized under a Developer ID separate from the upstream project, tracked with its own marketing version suffix (`v1.9.1.ck.1`).

**v1.9.1.ck.1**

* Fixed TV episodes in season 0 (specials) failing to auto-match against the metadata search API. The bundled Perl filename parser (`Utilities/ParseFilename/lib/Video/Filename.pm`) stripped leading zeros from the parsed season/episode with `s/^0+//`, which collapsed an all-zero value (`"00"`, or a bare `"0"`) to an empty string instead of `"0"`. Combined with `String.split(separator:)` in `MetadataHelper.swift` dropping that now-empty line by default, the parser's output no longer looked like a valid TV match and season-0 files silently fell back to being searched as movies. The zero-strip now only fires when a digit follows it, so an all-zero value collapses to `"0"` instead of `""`.
* Fixed TV auto-match failing when the parsed show name carries a trailing disambiguation year, e.g. `Masters of Sex (2013)`. `TheTVDB.swift` and `TheMovieDB.swift` now always search with the name as-is first, so legitimately year-disambiguated titles (reboots/remakes TheTVDB itself names that way, e.g. `Battlestar Galactica (2004)`) keep matching correctly, and only retry with the trailing `(YYYY)` stripped when the as-is search returns no results.
