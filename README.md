# selfExtractor

Found this useful script on linux journal
https://www.linuxjournal.com/node/1005818

Just adding to my git to easily use 
Need to add payload directory installer/payload/
cant push a empty folder

  <article data-history-node-id="1005818" role="article" about="/node/1005818" class="microblog full clearfix">

  
    

      <footer>
      <article typeof="schema:Person" about="/users/jeffparent">
  </article>

      <div class="author">
              by <span><a title="View user profile." href="/users/jeffparent" lang="" about="/users/jeffparent" typeof="schema:Person" property="schema:name" datatype="">jeff.parent</a></span>
 on December 6, 2007        
      </div>
    </footer>
  
  <div class="content">
    <ul class="links inline list-inline"><li class="comment-forbidden"></li></ul>
            <div class="field field--name-body field--type-text-with-summary field--label-hidden field--item">In this post I'll show you how to create a self extracting bash script to 
automate the installation of files on your system.  This script requires 
coreutils (for cat, tail), awk, gzip, tar and bash. There are 3 parts to the 
Base Self-Extracting Script
<ul>
    <li>The Payload</li>
    <li>The Decompression Script</li>
    <li>The Builder Script</li>
</ul>
Lets begin by creating our working directory and all the script files we will
need. 
<pre>
jeff:~$ mkdir -vp installer/payload
mkdir: created directory `installer'
mkdir: created directory `installer/payload'
jeff:~$ cd installer/
jeff:~/installer$
</pre>

<strong>The Payload</strong>
The payload directory will contain....just that, the payload of your 
installer. This is the location where you'll put all your tar files, scripts,
binaries, etc that you'll want installed onto the new system. For this 
example I have a tar file containing some text files that I'll want to 
install into a folder that I create into my home directory. Here is the 
listing of the tar file.
<pre>
jeff:~$ tar tvf files.tar
-rw-r--r-- jeff/jeff        40 2007-12-06 07:53 ./File1.txt
-rw-r--r-- jeff/jeff        92 2007-12-06 07:55 ./File2.txt
jeff:~$
</pre>

Now we must create the installation script that will handle the payload. 
This script contains any actions that you'd wish to be performed on the 
installation system, make directories, uncompress files, run system commands,
etc. For the example I will create a directory and untar the files into it.


<pre>
#!/bin/bash
echo "Running Installer"
mkdir $HOME/files
tar ./files.tar -C $HOME/files
</pre>

Now that we have filled our payload directory with all the files we'd like 
to install and created the installation script to the the files in their
correct location, our directory structure should look like this:

<pre>
jeff:~$ find installer/
installer/
installer/payload
installer/payload/installer
installer/payload/files.tar
jeff:~$
</pre>

<strong>The Decompression Script</strong>
The Decompression Script does most of the work. The compressed archive of 
your payload directory will actually be appended onto this script.  When 
ran, the script will remove the archive, decompress it and execute the 
install script we had created in the previous section.

<pre>
#!/bin/bash
echo ""
echo "Self Extracting Installer"
echo ""

export TMPDIR=`mktemp -d /tmp/selfextract.XXXXXX`

ARCHIVE=`awk '/^__ARCHIVE_BELOW__/ {print NR + 1; exit 0; }' $0`

tail -n+$ARCHIVE $0 | tar xzv -C $TMPDIR

CDIR=`pwd`
cd $TMPDIR
./installer

cd $CDIR
rm -rf $TMPDIR

exit 0

__ARCHIVE_BELOW__
</pre>

Ok, now lets go through this script step by step. After the bit of printout 
at the begining, the first real line of work creates a temporary directory
for us to decompress our payload into initially before we install it.

<pre> export TMPDIR=`mktemp -d /tmp/selfextrac.XXXXXX` </pre>

The <pre>-d</pre> flag tells mktemp to create a directory rather than a
file. The parameter at the end, <pre>/tmp/selfextract.XXXXXX</pre> is a
template of the name of the directory we are going to create. The X's are 
replaced with random characters to generate a random name, just incase we 
happen to be installing two things at the same time.

The next line in the script,

<pre>ARCHIVE=`awk '/^__ARCHIVE_BELOW__/ {print NR + 1; exit 0; }' $0`</pre>

find the line number where the archive starts in the script. The first 
parameter given to awk <pre>/^__ARCHIVE_BELOW__/</pre> tells it to search 
for a regular expression "A line starting with the characters 
'__ARCHIVE_BELOW__'". Each line is read and until the regular expression is 
satisfied. The next parameter <pre>{print NR + 1; exit 0; }</pre> tells 
awk to print out the number of records (number of lines read) plus 1, then 
quit. The third parameter <pre>$0</pre> is the first argument when running
this script, which happens to be the script's name ($0 = ./decompress).

We will now seperate the archive from the script and decompress it into the
temporary directory we have created.
<pre>tail -n+$ARCHIVE $0 | tar xzv -C $TMPDIR</pre>

tail prints out the end of a file. The parameter <pre>-n+$ARCHIVE</pre> 
tells tail to start at line number we just read in the previous command, and
print til the end of the file. <pre>$0</pre> again is the name of this 
script file. This output is then piped to tar where it is <pre>z</pre> 
ungzipped, and <pre>x</pre> unarchived into <pre>-C $TMPDIR</pre> the 
temporary directory.

The next section, we remember our current directory, <pre>CDIR=`pwd`</pre>
and step into the temporary directory <pre>cd $TMPDIR</pre>. From here we
run the script we created in the Payload section. After the script has been 
executed, we revert back to our previous directory <pre>cd $CDIR</pre> and
remove the temporary directory <pre>rm -rf $TMPDIR</pre>

The last two lines in this script are very important. First the line

<pre>exit 0</pre>

causes the script to stop executing. If we forget this line BASH would try to
execute the binary archive attached at the bottom and would cause problems. 
The very last line 

<pre>__ARCHIVE_BELOW__</pre>

tells awk that the binary archive starts on the next line. Make sure that 
this is the last line, no extra empty lines below this one.

Now that we have finished creating the Decompression script our directory 
should looke like this:

<pre>
jeff:~$ find installer/
installer/
installer/build
installer/payload
installer/payload/installer
installer/payload/files.tar
installer/decompress
jeff:~$
</pre>


<strong>The Builder Script</strong>
The last section of this installer is the script that builds the 
self-extracting script. This script compresses the payload and then adds the
decompresion script to the archive.

<pre>
#!/bin/bash
cd payload
tar cf ../payload.tar ./*
cd ..

if [ -e "payload.tar" ]; then
    gzip payload.tar

    if [ -e "payload.tar.gz" ]; then
        cat decompress payload.tar.gz > selfextract.bsx
    else
        echo "payload.tar.gz does not exist"
        exit 1
    fi
else
    echo "payload.tar does not exist"
    exit 1
fi

echo "selfextract.bsx created"
exit 0
</pre>

This script archives the payload directory 
<pre>tar cf ../payload.tar ./*</pre> and compresses it using gzip 
<pre>gzip payload.tar</pre>. Next the script concatenates the decompress
script with the compressed payload 
<pre>cat decompress payload.tar.gz > selfextract.bsx</pre>.


With all the scripts completed our directory should look like this:

<pre>
jeff:~$ find installer/
installer/
installer/build
installer/payload
installer/payload/installer
installer/payload/files.tar
installer/decompress
jeff:~$
</pre>


<strong>Test it out</strong>
Now that we have created everything, we can run the scripts and test it all
out.  Firstly run the build script. Your output should look as follows:

<pre>
jeff:~/installer$ ./build
selfextract.bsx created
jeff:~/installer$ ls
build  decompress  payload  payload.tar.gz  selfextract.bsx
jeff:~/installer$
</pre>


Now we have our bash script with the archive attached (selfextract.bsx). Run
this script and you should see the following output:

<pre>
jeff:~/installer$ ./selfextract.bsx

Self Extracting Installer

./files.tar
./installer
Running Installer
./File1.txt
./File2.txt
jeff:~/installer$ find /home/jeff/files
/home/jeff/files
/home/jeff/files/File1.txt
/home/jeff/files/File2.txt
jeff:~/installer$
</pre>


If you would like the installer to install something else, just place the 
files in the payload directory and modify the installer script to perform
the proper action.</div>
      
  </div>
</article>
