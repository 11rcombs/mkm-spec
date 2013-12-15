NOTICE: This specification is an inital draft, and is currently subject to
significant change at any time. Do not implement this format in a release
version of any player until a finalized specification is ready.

Matroska External Streams Manifest Files
========================================

MKV ordered chapters currently don't work unless the player can get a list of
potential referenced files (often every other file in the same directory as the
base file, sometimes recursively searched) and open each one to check its UUID.

This project aims to improve on this situation.

Ordered Chapter Overview
------------------------

MKV ordered chapters allow an MKV file to reference other MKV files, which can
contain parts of the timeline, often to help de-duplicate video and audio data
across multiple episodes in a series; it's perhaps best-known for its use in
fansubbed anime.

Unfortunately, the ordered chapters specification only includes the UUID of
each referenced file, and no information on how to locate the file in question,
such as a URL, a relative path, or a filename. This decision means that files
can be renamed without breaking compatibility, but forces players to do quite a
bit more work in locating the external referenced files.

Solution
--------

This project aims to create a simple text-based file format specification that
tells a player how to locate the segment files for a given MKV file or set of
files. The format acts similarly to a playlist, and a fully-specified file can
be referenced in a playlist file like any other media or playlist file.

Format Details
--------------
Matroska External Streams Manifest files use the file extension ".mkm", and
the MIME type "text/matroska-manifest". They are also identifiable by their
file signature: the first 18 bytes of a Matroska External Streams Manifest
file are always the ASCII string 'MATROSKA MANIFEST'.

Matroska External Streams Manifest files are always encoded using UTF-8.

Format Example
--------------

	MATROSKA MANIFEST
	base /absolute/path/to/index.mkv
	segment http://example-cdn.net/fully/qualified/url.mkv C234EECF8B16558FA2F18530E318062D
	segment ../path/relative/to/manifest.mkv 234C6AB6F5267EF79C65110CBF1536CA
	mode relative base
	segment resources/opening.mkv # Opener for episodes 1-4
	include ../../endings.mkm

The file above is appropriate for use with all relevant files hosted on HTTP
servers. Let's assume the files are all hosted on the server at
http://example.com/, and that the manifest file is at /data/media/test.mkm:

- The base file is loaded from http://example.com/absolute/path/to/index.mkv
- The player notes that if it encounters a chapter in index.mkv (or any of its dependencies) with the UUID C234EECF8B16558FA2F18530E318062D, that the file in question is located at http://example-cdn.net/fully/qualified/url.mkv
- The player similarly notes that the file with UUID 234C6AB6F5267EF79C65110CBF1536CA is at http://example.com/data/path/relative/to/manifest.mkv
- Upon encountering the line "mode relative base", the parser switches from resolving relative paths relative to the manifest to resolving them relative to the base file
- The player notes that if the base file requires any external chapter files whose UUIDs were not specified while parsing .mkm files, it should open http://example.com/absolute/path/to/resources/opening.mkv
- The parser ignores the #-delimited comment, and the trailing whitespace before it.
- The parser fetches http://example.com/absolute/endings.mkm and parses it for additional information, starting with paths defaulting to resolve relative to http://example.com/absolute/endings.mkm

General points
--------------
- This specification will attempt to prevent any ambiguities from being possible in MKM files, but if the parser detects a situation where correct behavior is not defined by this document, it MUST stop parsing immediately and fall back to playing the base file with dependencies resolved using the player's default internal mechanism. If the error occurs before the base file has been specified, the player MUST refuse to play any media corresponding to the MKM document being parsed. The default behavior being a hard failure is meant to discourage loosely-correct files that may play correctly in one compliant player, but not in another
- If out-of-band metadata, such as HTTP headers, specifies a character encoding for an MKM file, this data MUST be ignored. Compliant MKM files are always encoded in UTF-8
- If a path specified in an MKM file includes an ASCII space, tab, line feed, or hash ('#') character, it MUST be URL-encoded (e.g. '%20')
- Lines in MKM files are always separated by newline characters. Carriage return characters will not be interpreted as "special" or "whitespace" characters
- MKM parsing is case-sensitive
- If a relative path must be resolved relative to *base*, and *base* is null, abort parsing.

Parser Algorithm
----------------
The following steps are followed by a player in order to parse an MKM file:

1. Let *manifest URL* be the absolute URL representing the manifest.
2. Let *relative path mode* be *manifest*.
3. Let *UUID map* be an initially-empty associative array, with 16-byte UUIDs as keys and absolute URLs as values.
4. Let *additional file list* be an initially-empty list of absolute URLs to check for chapters with unknown locations.
5. If this manifest is being parsed in order to find dependency files for an MKV file whose location is already known, let *base path* be that file's URL. Otherwise, let *base path* be null.
6. Let *input* be the decoded text of the manifest's byte stream.
7. Let *position* be a pointer into input, initially pointing at the first character.
8. If the characters starting from *position* are "MATROSKA", followed by a U+0020 SPACE character, followed by "MANIFEST", then advance *position* to the next character after those. Otherwise, this isn't a Matroska manifest: if the base file's URL is already known, play that file, resolving dependencies using the player's internal method; otherwise, treat the error like an unparseable or corrupted playlist or media file; abort this algorithm.
9. If the character at *position* is neither a U+0020 SPACE character, a U+0009 CHARACTER TABULATION (tab) character, nor a U+000A LINE FEED (LF) character, then this isn't a Matroska manifest; abort this algorithm as specified above.
10. Collect a sequence of characters that are not U+000A LINE FEED (LF) characters, and ignore those characters. (Extra text on the first line, after the signature, is ignored.)
11. *Start of line*: If position is past the end of input, then jump to the last step. Otherwise, collect a sequence of characters that are U+000A LINE FEED (LF), U+0020 SPACE, or U+0009 CHARACTER TABULATION (tab) characters, and ignore those characters.
12. Now, collect a sequence of characters that are *not* U+000A LINE FEED (LF) characters, and let the result be *line*.
13. If *line* contains a U+0023 NUMBER SIGN character, drop the portion of *line* from the first NUMBER SIGN character to the end.
14. Drop any trailing U+0020 SPACE and U+0009 CHARACTER TABULATION (tab) characters at the end of *line*.
15. If *line* is the empty string, then jump back to the *step* labeled "start of line".
16. If *line* starts with "mode", followed by one or more U+0020 SPACE and U+0009 CHARACTER TABULATION (tab) characters, followed by "relative", followed by one or more U+0020 SPACE and U+0009 CHARACTER TABULATION (tab) characters, followed by "base" or "manifest", set *relative path mode* accordingly, then jump back to the *step* labeled "start of line".
17. If *line* starts with "base", skip one or more U+0020 SPACE and U+0009 CHARACTER TABULATION (tab) characters, then resolve the remainder of *line*  relative to *manifest URL*, let *base path* be the result, then jump back to the *step* labeled "start of line".
18. If *line* starts with "include", skip one or more U+0020 SPACE and U+0009 CHARACTER TABULATION (tab) characters, then resolve the remainder of *line* relative to the URL specified by *relative path mode*, fetch the specified file, and parse it using these steps, providing the parser with *base*. If the file cannot be fetched, or fails to parse, ignore the line; do not abort these steps. If the file parses correctly, add the results to *additional file list* and *UUID map*; do not modify *base*, and replace existing values in *UUID map* if necessary. Jump jump back to the *step* labeled "start of line".
19. If *line* starts with "segment", skip one or more U+0020 SPACE and U+0009 CHARACTER TABULATION (tab) characters, then separate the remainder of *line* into a list of *tokens*, separated by one or more U+0020 SPACE or U+0009 CHARACTER TABULATION (tab) characters. If there is only one token, resolve it relative to the URL specified by *relative path mode* and add the result to *additional file list*. If there are two or more tokens, resolve the first relative to the URL specified by *relative path mode* and parse the second as a hexadecimal number representing a 16-byte UUID (the string need not be zero-padded to 32 characters). Ignore additional tokens (for forwards-compatibility). Add an item *UUID map*, with the parsed UUID as the key and the resolved URL as the value, then jump back to the *step* labeled "start of line".
20. If none of the above conditions describe *line*, ignore it (for forwards-compatibility) and jump back to the *step* labeled "start of line".
21. Return the *UUID map*, the *additional file list*, and the *base path*.
