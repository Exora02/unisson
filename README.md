# Unisson

One music player for everything you listen to: local files (MP3/FLAC), Qobuz (hi-res),
YouTube Music and Spotify, merged into a single library. Built with Flutter; runs on
Android and Linux today.

Serverless by design: no Music Assistant server, each provider talks to its service directly.

## Status

Working prototype (v0.5). What works today:

- Search and playback from local files, Qobuz, YouTube Music and Spotify
- Merged library: one card per album/track across sources, with source priority and badges
- Per-play source and quality switching, with quality labels (bit depth, sample rate, bitrate)
- Queue, background playback and media-session controls
- Library persistence in SQLite: favourites, playlists, recently played
- Playlist and favourites import from Qobuz, YouTube Music and Spotify
- Local stream proxy between the player and the CDNs
- CI on GitHub Actions: `flutter analyze`, unit tests, release APKs per ABI

Not done yet: the bit-perfect Rust audio engine, Windows/macOS/iOS builds, Tidal.

## Stack

| Layer | Choice | Notes |
|---|---|---|
| UI | Flutter | single codebase |
| Playback | just_audio + audio_service | lossless decode, background playback, media session |
| Qobuz | port of the Music Assistant provider logic (Dart) | Qobuz API, subscriber credentials |
| YouTube Music | pure-Dart WEB_REMIX search + InnerTube stream resolution | client ladder with anonymous fallbacks; ytmusicapi and audios_resolver (MIT) as references |
| Spotify | PKCE login + Web API; playback through the Web Playback SDK in a webview | needs Spotify Premium |
| Local | pure-Dart tag reader, recursive directory scan | FLAC (Vorbis comments), MP3 (ID3v2) |
| Library | LibraryService + SQLite (sqflite) | merged search, source priority, favourites, playlists, history |
| Audio v2 (planned) | Rust cpal exclusive engine | bit-perfect: WASAPI exclusive / ALSA direct / CoreAudio hog / AAudio |

## Provider abstraction

```dart
abstract class MusicProvider {
  String get id;          // 'local' | 'qobuz' | 'ytm' | 'spotify'
  bool get isConfigured;
  bool get hasQualityTiers => false;  // true when the source has hi-res / CD / lossy tiers

  Future<SearchResults> search(String query);
  Future<StreamSpec> resolveStream(Track track, QualityPref pref);
}
```

StreamSpec = { uri, contentType, bitrate, sampleRate, bitDepth, expiresAt, + metadata that only
arrives with the stream }. QualityPref = highest | balanced | lowest. Each provider picks its best
format tier.

## Playback quality ladder

- Qobuz: FLAC up to 24/192 (subscription tier), fallback 320 kbps MP3
- YouTube Music: opus 251 (~150 kbps) > m4a 140 (128 kbps), no hi-res on YouTube
- Spotify: Web Playback SDK stream (bitrate shown when known)
- Local: original file, no transcode

## Source selection

Every album/track resolves against all configured providers into one merged card.
Playback picks automatically (priority local > Qobuz > YouTube Music) or the user picks the
source and quality per play. Mixed-source queues are supported.

## Build

```bash
flutter pub get
flutter run      # Android device or Linux desktop
flutter test
```

Tagged versions (`v*`) attach the release APKs to a GitHub release.

## Roadmap

- [x] YouTube Music feasibility spike (search, stream resolution)
- [x] Flutter scaffold + provider interface + YouTube Music search/play
- [x] Local files provider + merged library (source priority + badges)
- [x] Qobuz provider (auth, search, hi-res streams)
- [x] Playback: queue, background audio, source + quality switching
- [x] Library persistence: favourites, playlists, recently played
- [x] Import from Qobuz, YouTube Music and Spotify
- [x] Spotify provider (PKCE login, search, import, Web Playback SDK)
- [ ] Rust cpal exclusive playback engine (desktop first)
- [ ] Windows and macOS builds, iOS port
- [ ] Tidal provider

## References (spec material)

- music-assistant/server providers/qobuz + ytmusic (Apache-2.0)
- ytmusicapi (MIT): YTM web-client API reference
- audios_resolver (MIT): InnerTube stream resolution reference (ported)
- InnerTune/RiMusic (GPL): Android-client API extraction reference
- qbz (MIT, Rust): bit-perfect Qobuz desktop reference
- media_kit / libmpv: audio engine for bit-perfect evolution
- cpal: exclusive audio backends
