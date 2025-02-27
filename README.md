<!-- # Disclaimer -->
<!--
Downloading content from the internet for personal use (not distribution) is not illegal
(criminal law or copyright infringement); but by using this script, you are breaking
Youtube's Terms of Service (civil law). Then again, that's a problem for the
`youtube-dl` devs, not us.
-->
<!--
[![Donate](https://img.shields.io/badge/Donate-PayPal-green.svg)](lukelbd@gmail.com)
-->

# Overview

This script uses `youtube-dl` to download and save audio from youtube into `aac`/`m4a`
files (the native format of youtube audio), normalizes the audio/volume using
`ffmpeg-normalize`, and adds metadata tags using a python script called `ydm-metadata`.

**Note**: This project is a work-in-progress. A future version will be fully
OS agnostic and python-based, installed with ``pip install youtube-dl-music``,
and use the
[XDG base directory](https://wiki.archlinux.org/index.php/XDG_Base_Directory)
standard for configuration files.

# Installation

Currently, this script can only be used from UNIX shells -- i.e. MacOS, Ubuntu,
perhaps the Windows 10 Unix terminal. Install this library by navigating to your home
directory in the terminal and entering

    git clone https://github.com/lukelbd/youtube-dl-music

This should create a directory named `youtube-dl-music`.

Next you need to make sure the `ydm` and `ydm-metadata` executables are in your `$PATH`
variable. Simply add the line

    export PATH="$HOME/youtube-dl-music:$PATH"

to your shell configuration file, usually named `~/.bash_profile` or `~/.bashrc`, then
restart the shell or "source" the file with `source <file>`.

If you don't want to use `ydm-metadata`:

  * Manually install `ffmpeg`, `youtube-dl`, and `ffmpeg-normalize`.
  * Always use the flag `-q` when calling the `ydm` command.

If you do want to use `ydm-metadata`:

  * Make sure your version of `python3` is python3.6+. Check this with
    `python3 --version`. The `ydm-metadata` script has f-strings, which are a python
    3.6+ feature.
  * Run the `ydm-install` command. This installs the python packages needed for
    `ydm-metadata` (see below) and creates a file named `.config` in the
    `youtube-dl-music` directory.
  * Create an account with [Discogs](https://www.discogs.com/users/create) and another
    account with
    [MusicBrainz](https://musicbrainz.org/register?uri=%2Fdoc%2FHow_to_Create_an_Account).
    After creating the Discogs account, click on the top-right profile image and to
    Settings --> Developer, then click the "Generate token" button. Discogs and
    MusicBrainz are the two major online discography databases, each with their own
    strengths and weaknesses, and each with their own public python APIs. Therefore,
    this program uses both!
  * Fill the `.config` file with your desired destination folder, your Discogs token,
    your MusicBrainz username, and your MusicBrainz password (make sure the password is
    not an important one).

# Dependencies

The only non-python dependency is [ffmpeg](https://github.com/FFmpeg/FFmpeg). This is a
batteries-included package for creating/modifying media files. I recommend using
[MacPorts](https://www.macports.org) to install using
`sudo port install ffmpeg +nonfree`.
The `+nonfree` flag is required to add the proprietary `libfdk_aac` AAC
encoding package, which is [superior](https://trac.ffmpeg.org/wiki/Encode/AAC) to the
equivalent free package.

The remaining dependencies are python packages. The below packages are all installed by
the `ydm-install` command and listed in [requirements.txt](requirements.txt).

  * [BeautifulSoup](https://pypi.python.org/pypi/beautifulsoup4): For parsing the
    "release group" HTML webpage and finding any linked discogs "master releases".
  * [Python Imaging Library](https://pypi.python.org/pypi/PIL): For warping cover art
    images to square.
  * [discogs_client](https://github.com/discogs/discogs_client): For interacting with
    Discogs database.
  * [ffmpeg-normalize](https://github.com/slhck/ffmpeg-normalize): Python package for
    normalizing volume, requires ffmpeg accessible from shell.
  * [musicbrainzngs](https://github.com/alastair/python-musicbrainzngs): For interacting
    with MusicBrainz database.
  * [mutagen](https://github.com/quodlibet/mutagen): For writing metadata.
  * [unidecode](https://pypi.python.org/pypi/Unidecode): For translating accented
    characters to ASCII.
  * [youtube-dl](https://github.com/rg3/youtube-dl): Script for downloading youtube
    media.

# Documentation

For details, try

```sh
ydm --help
ydm-metadata --help
```

Below is a broad overview.

## Download script

This is the main script for downloading music. Usage is as follows:

```bash
ydm [flags] URL ARTIST NAME - SONG TITLE  # inside the main directory
ydm [flags] URL ARTIST NAME / SONG TITLE  # inside an artist subdirectory
```

Everything after 'URL' is interpreted as part of the destination location.
The `ARTIST NAME - SONG TITLE` usage places the file loose inside your music folder.
The `ARTIST NAME / SONG TITLE` usage places the file inside a *subdirectory* specific
to the artist. This file naming convention is *necessary* for `ydm-metadata` to
tag the file.

**Important**: If  `ydm` stops working, it is often because `youtube.com` has changed
how they store video/audio. The `youtube-dl` developers are very active and usually will
release an updated version within a couple days. Just call `youtube-dl -U` or `pip
install --upgrade youtube-dl` (depending on how it was installed), and it should start
working again.

## Metadata script

The `ydm-metadata` script is called automatically by `ydm`, but you may want to use it or re-use it on existing files. Usage is as follows:

```bash
ydm-metadata [FLAGS] PATH1 [PATH2 ...]
```

This time the filename(s) must have escaped spaces. The following command-line options are available:

* `-v`: Increases verbosity.
* `-s`: Enables strict mode. Makes sure recording name matches input title name exactly.
* `-f`: Enables forget mode. This disables caching and looking up previous user responses.
* `-c`: Enables confirm mode. This prompts user to always confirm the release group and release for album artwork.

In some cases, the tagging algorithm fails with `-s` -- e.g. a search for "Aerosmith - Dude Looks Like A Lady" filters out titles "Dude (Looks Like A Lady)" with parentheses, because parentheses often contain additional information unrelated to the song title. But in other cases it may be necessary -- e.g. without `-s`, a search for "Pink Floyd - Mother" also returns "Pink Floyd - Matilda Mother".

`ydm-metadata` is certainly not the fastest tagging algorithm out there. For example, the builtin cover art downloader for the Android app "PowerAmp" is pretty darn fast. `ydm-metadata` is designed to strictly *minimize tagging errors*, and get the *highest possible quality cover art*. So it is slow, but very accurate.

## Metadata script algorithm

`Mutagen` is used to write tags. For `mp3` files, tags are added to the ID3 header, while for `aac` and `m4a` files, tags are added in some mysterious way that Apple pioneered (but should still be readable by most media players).

Here's a play-by-play of what `ydm-metadata` does:

1. Run strict search of MusicBrainz "recordings" matching the *filename-inferred* track
   name and with artist credits matching the *filename-inferred* artist name.
   * Choose the first credit artist and ask for user input if the first credit artists
     in the release list differ.
   * Permit names starting *with or without* "the" (e.g. "Animals" instead of "The
     Animals"). If you omit the "the", the music in your filesystem will be sorted more
     naturally.
   * Permit names ending with "&" or "and" something (e.g. "Tom Petty *and the
     Heartbreakers*").
   * Write "artist" and "title" metadata according to the search results.

2. Verify the recordings returned, eliminate some matches but permit others.
   * Ensure "meaningful words" in the recording name matches the filename-inferred song
     name. For example, in a search for "Mother", "Atom Heart Mother" and "Matilda
     Mother" are ignored.
   * Ignore content inside parentheses. For example, in a search for "The Wall", "The
     Wall (part 1)" and "The Wall (part i)" are included.
   * Permit optional matches for text on either side of a "/". For example, in a search
     for "Hush", permit the recording name "Hush / I'm Alive".

3. Get release "groups" from the release list belonging to each recording, and
   consolidate the recordings (sorted by unique ID) into their corresponding release
   groups.
   * Sort the release groups to prefer albums and singles over compilations and live
     performances.
   * Try to pick release groups with releases from *earlier years* rather than later
     years. These are more likely to be "original" versions.

4. Get several *album related* metadata categories from our ordered hierarchy of
   releases belonging to unique release groups. This is the most important step.
   * Write "year" metadata from the earliest release amongst all members of *all*
     release groups.
   * Write "album" metadata from the highest-ranked release group containing the
     earliest release years.
   * Write "genres" from the first release group in the hierarchy for which genres are
     available. Genres are obtained from both MusicBrainz and the Discogs "master"
     recording (found by searching the MusicBrainz HTML webpage for a Discogs URL).
     MusicBrainz genres are translated and filtered to the limited subset used by
     Discogs.
   * Write "cover art" metadata from the latest release and most *modern* release
     format amongst releases in the highest-ranked release group. This gets the
     nicest-looking cover art available. If `--confirm` was passed, you can choose to
     bypass certain release groups and releases.

