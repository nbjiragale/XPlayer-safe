# Evidence from the emulator test

Android 11 (API 30), x86_64, software emulation, ARM translation. See REPORT.md section 9.

| File | What it shows |
|---|---|
| 01_main_library_screen.png | The app's main screen after a clean start (no crash) |
| 02_video_thumbnail_decoded.png | The app decoded a frame from my test clip and read its 00:06 length |
| 03_main_build_playback_failed_load_timeout.png | The shipped main build in this slow emulator: "Playback failed" = the app's 10-second library-load watchdog |
| 04_player_tutorial_after_fix_of_test_timeout.png | Test copy with the watchdog stretched: the player loads and shows its first-run tutorial |
| 05_external_file_open_needs_all_files.png | Opening by file path from outside asks for All Files (Android 11 scoped storage, by design) |
| 06_library_play_done_progress_bar.png | After playing from the library: the thumbnail has the "watched" progress bar |
| 07_strict_build_library.png | The strict build (no all-files, no overlay) starts and shows the library |

`emulator_test_log.txt` has the key log lines, including the 265,344 audio frames (6.02 s) delivered by the app's own process.
