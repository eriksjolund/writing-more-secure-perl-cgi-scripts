# writing-more-secure-perl-cgi-scripts

How to write more secure Perl CGI scripts. A guide with some tips.
 
__Update March 2020__: Migrate this tutorial from 2008 to Github for archival purposes. (Nowadays there are better programming languages and web frameworks).

## Introduction

This guide contains some example CGI scripts that demonstrates how to improve security in CGI scripts. The example _count7.cgi_ has the best security.

## Advices

If you are writing your CGI scripts in perl, you should

use the `-T` flag. It is also a good idea to use the `-w` flag. The script will hence start with

```
#!/usr/bin/perl -Tw 
```


Avoid using temporary files as much as possible. If you want to call an external program, try to use the `open2()` call. This works under the assumption that the external program can read the input data from standard input and write the result to standard output. For instance, like this

```
use IPC::Open2;
my $childpid = open2(\*READ, \*WRITE, $config::blastall_path , "-m", "7", "-d" , $db, "-p", "blastp", "-b", $max_hits, "-v", $max_hits, "-M", "BLOSUM62"   )
 or die "Could not open pipe: $!";

print WRITE $blastinput;
close(WRITE);

my $blastoutput;
while(<READ>) {
  $blastoutput .= $_;
}
close READ;
waitpid($childpid, 0);
```

never use something like this

```
system("myprogram -f $cgiparam");
```

instead use the comma separated syntax

```
system("myprogram","-f","$cgiparam");
```

## Example CGI scripts


### count.py

_count.py_ is a Python script that is used by all of the example cgi scripts.


```
#!/usr/bin/python

import sys

if len(sys.argv) == 2:
  sys.exit()
elif len(sys.argv) == 3:
  f=open(sys.argv[2], "r") 
else: 
  sys.exit()

#f.seek(0)

lines=0
bytes=0
for line in f:
  lines += 1
  bytes += len( line ) 

if (sys.argv[1] == "bytes"): 
  print str(bytes)

if (sys.argv[1] == "lines"): 
  print str(lines)
```

### count1.cgi

Configuring __apache httpd__ to allow _Server Side Includes_ ( SSI ) is dangerous. The option is called `Includes`.

A hacker could upload a file with SSI exec statements. Those commands would then be executed as soon as a web browser tries to access the uploaded file. Often SSI is only activated for filenames ending with "_.shtml_". In the this CGI script the web surfer chooses the file name and can choose a file name ending with "_.shtml_".

```
#!/usr/bin/perl 

use CGI qw(:standard);
use CGI::Carp qw(fatalsToBrowser);

my $cgi = new CGI;
if ($cgi->cgi_error())  { die() };

print
  $cgi->header(),
  $cgi->start_html( '' ),
  $cgi->h1('Count'),       
  $cgi->start_form(-method=>'GET'),
  "type:",
  $cgi->popup_menu(-name=>'type',-values=>['lines','bytes']),
  $cgi->p, 
  "filename",
  $cgi->textfield(-name=>'fname'),
  $cgi->p,
  $cgi->textarea(-name=>'content', -rows=>10,  -columns=>50),
  $cgi->p,
  $cgi->submit(),
  $cgi->end_form,
  $cgi->hr;

if ($cgi->param) {

    my $fname = $cgi->param('fname');
    my $content = $cgi->param('content');
    my $type = $cgi->param('type'); 

    my $filename="tmp/" . $fname;

    my $outfh;
    open($outfh, ">", $filename) 
        or die "Couldn't open $tmpfile for writing: $!";

    print $outfh $content;
    close $outfh;

    my $result=`/home/esjolund/public_html/cgi-bin/count.py $type $filename`;

    print "The file <b>$fname</b> has $result $type";
    print $cgi->end_html;  
}
```

### count2.cgi

Examplifies `use File::Temp qw(tempfile);`. If you can't avoid using temporary files, please use this perl module to create them. Although not possible here, you should preferrably use `UNLINK => 1` as argument to the `tempfile()` function.

```
#!/usr/bin/perl 

use CGI qw(:standard);
use CGI::Carp qw(fatalsToBrowser);
use File::Temp qw(tempfile);

my $cgi = new CGI;
if ($cgi->cgi_error())  { die() };

print
  $cgi->header(),
  $cgi->start_html( '' ),
  $cgi->h1('Count'),       
  $cgi->start_form(-method=>'GET'),
  "type:",
  $cgi->popup_menu(-name=>'type',-values=>['lines','bytes']),
  $cgi->p,
  $cgi->textarea(-name=>'content', -rows=>10,  -columns=>50),
  $cgi->p,
  $cgi->submit(),
  $cgi->end_form,
  $cgi->hr;

if ($cgi->param) {

    my $content = $cgi->param('content');
    my $type = $cgi->param('type'); 

    File::Temp->safe_level( File::Temp::HIGH );
    my ( $outfh, $filename ) =
       tempfile( "tmp/myfiles.XXXXXX", UNLINK => 0 );
    if ( !defined $outfh ) {
	die "error: no temporary file\n";
    }

    print $outfh $content;
    close $outfh;

    my $result=`/home/esjolund/public_html/cgi-bin/count.py $type $filename`;

    print "Content has $result $type";
    print $cgi->end_html;  
}
```


### count3.cgi

It is often possible to avoid temporary files by using pipes. In Perl this is done with `open2()`

```
#!/usr/bin/perl 

use CGI qw(:standard);
use CGI::Carp qw(fatalsToBrowser);
use IPC::Open2;

my $cgi = new CGI;
if ($cgi->cgi_error())  { die() };

print
  $cgi->header(),
  $cgi->start_html( '' ),
  $cgi->h1('Count'),       
  $cgi->start_form(-method=>'GET'),
  "type:",
  $cgi->popup_menu(-name=>'type',-values=>['lines','bytes']),
  $cgi->p,
  $cgi->textarea(-name=>'content', -rows=>10,  -columns=>50),
  $cgi->p,
  $cgi->submit(),
  $cgi->end_form,
  $cgi->hr;

if ($cgi->param) {

    my $content = $cgi->param('content');
    my $type = $cgi->param('type'); 

    my($child_out, $child_in);
    $pid = open2($child_out, $child_in, "/home/esjolund/public_html/cgi-bin/count.py", $type,"/dev/stdin");
    print $child_in $content;
    close($child_in);
    my $result=<$child_out>;  
    waitpid($pid,0);

    print "Content has $result $type";
    print $cgi->end_html;  
}
```


### count4.cgi

The previous example `count3.cgi`  is insecure as it passes the `$type` variable as directly to the command line ( to a shell ). We should instead use the comma separated syntax of `open2()` as shown here:

```
#!/usr/bin/perl 

use CGI qw(:standard);
use CGI::Carp qw(fatalsToBrowser);
use IPC::Open2;

my $cgi = new CGI;
if ($cgi->cgi_error())  { die() };

print
  $cgi->header(),
  $cgi->start_html( '' ),
  $cgi->h1('Count'),       
  $cgi->start_form(-method=>'GET'),
  "type:",
  $cgi->popup_menu(-name=>'type',-values=>['lines','bytes']),
  $cgi->p,
  $cgi->textarea(-name=>'content', -rows=>10,  -columns=>50),
  $cgi->p,
  $cgi->submit(),
  $cgi->end_form,
  $cgi->hr;

if ($cgi->param) {

    my $content = $cgi->param('content');
    my $type = $cgi->param('type'); 

    my($child_out, $child_in);
    $pid = open2($child_out, $child_in, "/home/esjolund/public_html/cgi-bin/count.py",$type);
    print $child_in $content;
    close($child_in);
    my $result=<$child_out>;  
    waitpid($pid,0);

    print "Content has $result $type";
    print $cgi->end_html;  
}
```


### count5.cgi

Examplifies taint mode, the `-T` flag

```
#!/usr/bin/perl -T

use CGI qw(:standard);
use CGI::Carp qw(fatalsToBrowser);
use IPC::Open2;

use Scalar::Util qw(tainted);

my $cgi = new CGI;
if ($cgi->cgi_error())  { die() };

print
  $cgi->header(),
  $cgi->start_html( '' ),
  $cgi->h1('Count'),       
  $cgi->start_form(-method=>'GET'),
  "type:",
  $cgi->popup_menu(-name=>'type',-values=>['lines','bytes']),
  $cgi->p,
  $cgi->textarea(-name=>'content', -rows=>10,  -columns=>50),
  $cgi->p,
  $cgi->submit(),
  $cgi->end_form,
  $cgi->hr;

if ($cgi->param) {

    if (tainted($cgi->param('content'))) 
       { print 'variable content tainted!'; } 
    else 
       { print 'variable content not tainted!'; } 
    print `echo $cgi->param('content')`;

    my $content = $cgi->param('content');
    my $type = $cgi->param('type'); 

    my($child_out, $child_in);
    $pid = open2($child_out, $child_in, "/home/esjolund/public_html/cgi-bin/count.py",$type);
    print $child_in $content;
    close($child_in);
    my $result=<$child_out>;  
    waitpid($pid,0);

    print "Content has $result $type";
    print $cgi->end_html;  
}
```

### count6.cgi

If we try to run the previous example _count5.cgi_,  perl will complain that the `PATH` variable is insecure. If we use the `-T` flag, we need to set the `PATH` environment variable.

```
#!/usr/bin/perl -T

use CGI qw(:standard);
use CGI::Carp qw(fatalsToBrowser);
use IPC::Open2;

use Scalar::Util qw(tainted);

my $cgi = new CGI;
if ($cgi->cgi_error())  { die() };

$ENV{'PATH'} = '/bin:/usr/bin';
delete @ENV{'IFS', 'CDPATH', 'ENV', 'BASH_ENV'};

print
  $cgi->header(),
  $cgi->start_html( '' ),
  $cgi->h1('Count'),       
  $cgi->start_form(-method=>'GET'),
  "type:",
  $cgi->popup_menu(-name=>'type',-values=>['lines','bytes']),
  $cgi->p,
  $cgi->textarea(-name=>'content', -rows=>10,  -columns=>50),
  $cgi->p,
  $cgi->submit(),
  $cgi->end_form,
  $cgi->hr;

if ($cgi->param) {

    my $content = $cgi->param('content');
    my $type = $cgi->param('type'); 

    if ( tainted($type) ) 
       { print 'tainted!'; } 
     else 
       { print 'not tainted!'; } 

    print $cgi->hr;
    print `echo $type`;

    my($child_out, $child_in);
    $pid = open2($child_out, $child_in, "/home/esjolund/public_html/cgi-bin/count.py",$type);
    print $child_in $content;
    close($child_in);
    my $result=<$child_out>;  
    waitpid($pid,0);

    print "Content has $result $type";
    print $cgi->end_html;  
}
```

### count7.cgi

If we try to run the previous example _count6.cgi_  perl will complain that we are using untainted variables in an insecure way. To use CGI input variables we need to untaint them by processing them with a regular expression

```
#!/usr/bin/perl -T

use CGI qw(:standard);
use CGI::Carp qw(fatalsToBrowser);
use IPC::Open2;

use Scalar::Util qw(tainted);

my $cgi = new CGI;
if ($cgi->cgi_error())  { die() };

$ENV{'PATH'} = '/bin:/usr/bin';
delete @ENV{'IFS', 'CDPATH', 'ENV', 'BASH_ENV'};

print
  $cgi->header(),
  $cgi->start_html( '' ),
  $cgi->h1('Count'),       
  $cgi->start_form(-method=>'GET'),
  "type:",
  $cgi->popup_menu(-name=>'type',-values=>['lines','bytes']),
  $cgi->p,
  $cgi->textarea(-name=>'content', -rows=>10,  -columns=>50),
  $cgi->p,
  $cgi->submit(),
  $cgi->end_form,
  $cgi->hr;

if ($cgi->param) {

    my $type;
    if ( $cgi->param('type')=~/^(bytes|lines)$/ ) {
        $type = $1; 
    } else {
        print "variable type needs to be bytes or lines";
        exit(0);
    }

    my $content;
    if ( $cgi->param('content')=~/^(.*)$/s ) {
        $content = $1; 
    } else {
        exit(0);
    }

    if ( tainted($type) ) 
       { print 'tainted!'; } 
     else 
       { print 'not tainted!'; } 

    print $cgi->hr;
    print `echo $type`;

    my($child_out, $child_in);
    $pid = open2($child_out, $child_in, "/home/esjolund/public_html/cgi-bin/count.py",$type);
    print $child_in $content;
    close($child_in);
    my $result=<$child_out>;  
    waitpid($pid,0);

    print "Content has $result $type";
    print $cgi->end_html;  
}
```

