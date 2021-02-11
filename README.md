# stagit

static git page generator.

It generates static HTML pages for a git repository.

## Changes
This is my personal fork of stagit.    
It has been modified to fit my needs and some refs points
to cli apps on my web-server.

* Title of files while viewing is now a link to download the file
* The index page now links to the repository directory
* md4c has been added to parse markdown files
* Added classes and ids to some html tags to easier style them

The link to download the file points to `/raw/?repository=&file=` where file is the full path to the file relative to the repository. If branch is omitted, HEAD is assumed.

Make necessary changes to the web-server to serve the file    

To view a file in a bare repository:

    $cd path/to/repo
    $git show HEAD:path/to/file

Simple python cgi to serve the file:

```
#!/bin/python3

import cgi
import subprocess
import mimetypes
import sys

def getFile(repo, filepath, branch):
    gitcmd = subprocess.run(['git', 'show', f'{branch}:{filepath}'],
            cwd=f'/home/pi/repositories/{repo}',
            capture_output=True)
    return gitcmd.stdout


def guessType(filepath, extensions_map):
    ext = '.' + filepath.split('.')[-1]
    if ext in extensions_map:
        return extensions_map[ext]
    ext = ext.lower()
    if ext in extensions_map:
        return extensions_map[ext]
    return extensions_map['']


if not mimetypes.inited:
    mimetypes.init()
extensions_map = mimetypes.types_map.copy()
extensions_map[''] = 'application/octet-stream'

form = cgi.FieldStorage()
repo = form.getvalue('repository')
filepath = form.getvalue('file')
if (form.getvalue('branch')):
    branch = form.getvalue('branch')
else:
    branch = 'HEAD'

ctype = guessType(filepath, extensions_map)
response = getFile(repo, filepath, branch)

if response:
    print(f"Content-Type: {ctype}")
    print(f"Conent-Length: {str(len(response))}")
    print()
    sys.stdout.flush()
    sys.stdout.buffer.write(response)
else:
    print("Content-Type: text/plain")
    print()
    print("404: File not found")

``` 

The index page linking to the directory will not change the default 
behavior of stagit if the html files have been created with the [provided 
shell script](example_create.sh.html), since log.html is symlinked
to index.html.    

This change was made so that the web-server could check if the
request is for a directory. If it is check if there is a file called 
README.md.html in the sub-directory file, if it exists serve that 
file instead of log.html.

Lighttpd conf to implement this:

    url.redirect = ( "^\/stagit\/([a-zA-Z0-9-_.,]\/$" => "/stagit/$1/file/README.md" )
    url.rewrite-if-not-file = ( "^\/stagit/([a-zA-Z0-9-_,.]*)\/file\/README.md.html$" => /redirect/$1" )
    url.redirect += ( "^\/redirect/\([a-zA-Z0-9-_,.]*)" => "/stagit/$1/log.html" )

### md4c
md4c is used to parse and convert markdown documents into html. If a file 
ends with `.md` it will be parsed by md4c rather than the regular way 
stagit parses plain text files.

https://github.com/mity/md4c

## Usage

Make files per repository:

    $ mkdir -p htmldir && cd htmldir
    $ stagit path-to-repo

Make index file for repositories:

    $ stagit-index repodir1 repodir2 repodir3 > index.html


## Build and install

    $ make
    # make install


## Dependencies

- C compiler (C99).
- libc (tested with OpenBSD, FreeBSD, NetBSD, Linux: glibc and musl).
- libgit2 (v0.22+).
- POSIX make (optional).


## Documentation

See man pages: stagit(1) and stagit-index(1).


### Building a static binary

It may be useful to build static binaries, for example to run in a chroot.

It can be done like this at the time of writing (v0.24):

    cd libgit2-src

    # change the options in the CMake file: CMakeLists.txt
    BUILD_SHARED_LIBS to OFF (static)
    CURL to OFF              (not needed)
    USE_SSH OFF              (not needed)
    THREADSAFE OFF           (not needed)
    USE_OPENSSL OFF          (not needed, use builtin)
    
    mkdir -p build && cd build
    cmake ../
    make
    make install


### Extract owner field from git config

A way to extract the gitweb owner for example in the format:

	[gitweb]
		owner = Name here

Script:

    #!/bin/sh
    awk '/^[ 	]*owner[ 	]=/ {
        sub(/^[^=]*=[ 	]*/, "");
        print $0;
    }'


### Set clone url for a directory of repos

    #!/bin/sh
    cd "$dir"
    for i in *; do
        test -d "$i" && echo "git://git.codemadness.org/$i" > "$i/url"
    done


### Update files on git push

Using a post-receive hook the static files can be automatically updated.
Keep in mind git push -f can change the history and the commits may need
to be recreated. This is because stagit checks if a commit file already
exists. It also has a cache (-c) option which can conflict with the new
history. See stagit(1).

git post-receive hook (repo/.git/hooks/post-receive):

    #!/bin/sh
    # detect git push -f
    force=0
    while read -r old new ref; do
        hasrevs=$(git rev-list "$old" "^$new" | sed 1q)
        if test -n "$hasrevs"; then
	        force=1
            break
        fi
    done

    # remove commits and .cache on git push -f
    #if test "$force" = "1"; then
    # ...
    #fi

    # see example_create.sh for normal creation of the files.


### Create .tar.gz archives by tag

    #!/bin/sh
    name="stagit"
    mkdir -p archives
    git tag -l | while read -r t; do
    	f="archives/${name}-$(echo "${t}" | tr '/' '_').tar.gz"
    	test -f "${f}" && continue
    	git archive \
    		--format tar.gz \
    		--prefix "${t}/" \
    		-o "${f}" \
    		-- \
    		"${t}"
    done


## Features

- Log of all commits from HEAD.
- Log and diffstat per commit.
- Show file tree with linkable line numbers.
- Show references: local branches and tags.
- Detect README and LICENSE file from HEAD and link it as a webpage.
- Detect submodules (.gitmodules file) from HEAD and link it as a webpage.
- Atom feed log (atom.xml).
- Make index page for multiple repositories with stagit-index.
- After generating the pages (relatively slow) serving the files is very fast,
  simple and requires little resources (because the content is static), only
  a HTTP file server is required.
- Usable with text-browsers such as dillo, links, lynx and w3m.


## Cons

- Not suitable for large repositories (2000+ commits), because diffstats are
an expensive operation, the cache (-c flag) is a workaround for this in 
some cases.
- Not suitable for large repositories with many files, because all files 
are written for each execution of stagit. This is because stagit shows the 
lines of textfiles and there is no "cache" for file metadata 
(this would add more complexity to the code).
- Not suitable for repositories with many branches, a quite linear 
history is assumed (from HEAD).

  In these cases it is better to just use cgit or possibly change stagit to
  run as a CGI program.

- Relatively slow to run the first time (about 3 seconds for sbase, 
1500+ commits), incremental updates are faster.
- Does not support some of the dynamic features cgit has, like:
  - Snapshot tarballs per commit.
  - File tree per commit.
  - History log of branches diverged from HEAD.
  - Stats (git shortlog -s).

  This is by design, just use git locally.
