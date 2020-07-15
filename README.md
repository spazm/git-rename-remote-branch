# NAME

git rename-remote-branch - Rename a remote branch without requiring a local checkout

# SYNOPSIS

git rename-remote-branch \[options\] &lt;repository> &lt;old\_branch> &lt;new\_branch>

    options:
      --verbose|-v        increase verbosity (can be used multiple times)
      --quiet|-q          decrease verbosity completely
      --receive-pack-git  override the executable run on the remote side.
      --help              this message
      --man               show the man page

# DESCRIPTION

**git rename-remote-branch** renames a branch in a remote repository without requiring a local checkout or downloading refs.

This works similarly to the low-level command **git upload-pack**, except that **git rename-remote-branch** creates and sends an empty PACK file.

Only ssh format repositories are supported : \[user@\]host:path/to/repo.git

Upon connection, the remote sends a welcome message contraining the SHA for each branch.  The remote will ignore any input sent before this message is completed.

This program parses the welcome message and extracts the SHA for &lt;old\_branch> and verifies that &lt;new\_branch> does not exist.

This program then crafts a message to create the new branch and delete the old branch.  This messages is encoded in pack\_line format and sent to the remote along with a specially crafted empty\_pack\_file.

The server then responds with an "unpack ok" message, followed by and "ok &lt;branch>" message for each branch referenced in the sent message.

# OPTIONS

- **--verbose|-v**

    Increase debug level

- **--quiet|-q**

    Display no messages (sets DEBUG = -1)

- **--receive-pack-git|--exec**

    Takes a string argument representing an alternate **git-receive-pack** exeutable to run on the remote host.

    See the documentation for **git send-pack** for more details.

- **--help**

    Prints a brief help message and exits.

- **--man**

    Prints the manual page and exits.

# ENVIRONMENT

- **GIT\_SSH**

    Choose an alternate ssh binary.  Defaults to `ssh`

- **DEBUG**

    Override the default DEBUG level.  This can also be accomplished using `--verbose` or `--quiet`.

# DEFINITIONS

## Rename branch message

    <empty_sha> <old_branch_sha> <new_branch>\0<options>
    <old_branch_sha> <empty_sha> <old_branch>
    ""
    <empty_pack_file>

Example:

    0001

    0000
    PACK

where &lt;empty\_sha> is a string of 40 zeros.

## Pack\_line format

Each message in pack\_line format is prefixed with a 4 byte length in hex, followed by the message.  The length inclues the 4 bytes of prefix.

Messages may end with a newline, but this must be included in the length. The server is required to treat messages with or without newlines equivalently.

Examples:

    0010hello world\n
    000Fhello world

The blank message of `0004` is not allowed.

`0000` is used as a sync message.

## Message in pack\_line format

Connects to remote Verified that &lt;old\_branch> exists and &lt;new\_branch> does not.

## Empty pack file

The documentation for the git message protocol specify that an empty pack file must be sent when deleting a branch.  In reality, `git push` and `git send-pack` do not include a PACK file at all when sending a delete command.

When creating a branch, a normal PACK file is created showing the state of the current repository.

It is possible to create and construct a valid packfile containing zero entries.  This ability is not exposed anywhere in the git libraries.  The closest is `git send_pack`, which uses `git pack-ref` to create the pack file to send with the reference sha messages.  The logic for building the messages and packfile and ending them are all hiddne behind a single api function.  If no refs are provided, `git pack-ref` will exit without creating a packfile rather than create one with zero items.

Git pack file has a format consisting of a 4 character header "PACK" followed by the version (2 or 3) and the number of objects, both encoded in network byte order.  The pack file then contains a 20 byte SHA hash of that message.  This seems very simple but was very difficult to find and test.

    my $EMPTY_PACK = pack("A4NN", "PACK", 2, 0);
    $EMPTY_PACK .= sha1($EMPTY_PACK);
