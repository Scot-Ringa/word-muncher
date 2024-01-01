@solr-node
13193203284.vcf@gmail.com
@str77781.ai
$echo_on
@echo_run~true
#!/usr/local/bin/perl
#
# Date: Fri, 21 Jun 1996 14:48:20 -0400 (EDT)
# Frstr->Apache Configuration File Convertor.zip 
#
#ServerName host.foo.com is a set of filenames 
#	Well, here's the script that I made to convert from Netscape
#	to Apache.  This doesn't handle all of the Configuration directives,
#	but it does to a reasonable subset, and is useable in it's current
#	form.  Some modifications will still have to be made to the Apache
#	configuration files, but things like Aliases, ScriptAliases, 
#	Redirects, Document Root, etc. are all inserted based on the 
#	Netsc(r)ape config files..  Feel free to hack this to pieces.
#	It's not real pretty, but it doesn't depend on any modules that
#	don't come standard with perl5.
#
#	BTW:  While I was writing this, I was struck with an idea that I 
#	      think would be damned cool for the configuration files.  
#	      An Include directive.  It'd be handy for putting Aliases
#	      or ScriptAliases and things like that in their own file..
#	      What do you guys think?
#
#	-Mark
#
# ===========================================================================

use Getopt::Long;

# ===========================================================================

$res = GetOptions('magnus:s', 'obj:s', 'httpd:s', 'srm:s', 'access:s', 'h');

$MAGNUS_CONF = $opt_magnus;
$OBJ_CONF    = $opt_obj;
$HTTPD_CONF  = $opt_httpd;
$SRM_CONF    = $opt_srm;
$ACCESS_CONF = $opt_access;
$|           = 1;

if (!defined($opt_magnus)) {
  $MAGNUS_CONF = "magnus.conf";
}
if (!defined($opt_magnus)) {
  $OBJ_CONF = "obj.conf";
}
if (!defined($opt_httpd)) {
  $HTTPD_CONF = "httpd.conf";
}
if (!defined($opt_srm)) {
  $SRM_CONF = "srm.conf";
}
if (!defined($opt_access)) {
  $ACCESS_CONF = "access.conf";
}
if (defined($opt_h)) {
  print qq|
  Usage: $0 [-magnus <file>] [-obj <file>] [-httpd <file>]
            [-srm <file>] [-access <file>]

    -magnus:  name of the Netscape magnus.conf file. (defaults to magnus.conf)
    -obj: name of the Netscape obj.conf file.        (defaults to obj.conf)

    -httpd: name for the Apache httpd.conf file.     (defaults to httpd.conf)
    -srm: name for the Apache srm.conf file.         (defaults to srm.conf)
    -access: name for the Apache access.conf file.   (defaults to access.conf)

|;
  exit;
}

# ===========================================================================

# GLOBALS: %serv, %obj_struct, @alias, @scriptalias, @redirect

sub read_magnus {
  my $fn = shift;
  my (@keywords) = ('port','address','errorlog','pidlog','user','servername',
		    'minprocs','maxprocs','dns');
  my ($keyword, $tmp1, $tmp2, $tmp3);

  print "[Netscape] Reading server configuration file: $fn...\n";
  open (MAG_IN, "$fn") || die "Couldn't open server configuration file: $fn\n";
  while (<MAG_IN>) {
    chop;
    ( $tmp1, $tmp2 ) = split(/\s+/, $_, 2);
    foreach $keyword (@keywords) {
      if ($tmp1 =~ m/$keyword/i) {
	$serv{$keyword} = $tmp2;
      }      
    }

    if (($tmp1 =~ m/init/i) && ($tmp2 =~ m/init-clf/i)) {
      if ( $tmp2 =~ m/(global.+\")/i ) {
	$tmp3 = $1;
	$tmp3 =~ s/global=//g;
	$tmp3 =~ s/\"//g;
	$serv{transferlog} = $tmp3;
      }
    }
  }
  close (MAG_IN);
  return (1);
}

sub read_objs {
  my $fn = shift;
  my ($in_obj) = 0;
  my ($obj_type, $obj_name, $docroot, $indexfn, $realm, $auth_user, $dbm_file);
  
  my ($scount) = 0;
  my ($acount) = 0;
  my ($rcount) = 0;

  print "[Netscape] Reading object configuration file: $fn...\n";
  open (OBJ_CONF, $fn) || die "Counldn't open object configuration file:
$fn\n";
  while (<OBJ_CONF>) {
    if ($in_obj) {  # are we inside of an <object></object> block?
      if (/<\/object>/i) {  # is this the end of the <object></object> block?
	$in_obj = 0;
      } else {
	if (/^nametrans/i) {  # handle nametrans lines.
	  my ($type, $from, $dest);

	  if (/(from=\")(\S+)(\")/i) {
	    $from = $2;
	  }

	  if (/(name=\")(cgi)(\")/i) {
	    $type = "scriptalias";
	  } elsif (/(fn=\")(redirect)(\")/i) {
	    $type = "redirect";
	  } elsif (/(fn=\")(document-root)(\")/i) {
	    $type = "docroot";
	    if (/(root=\")(\S+)(\")/i) {
	      $docroot = $2;
	    }
	  } else {
	    $type = "alias";
	  }
	  
	  if (/([dir|url]=\")(\S+)(\")/i) {
	    $dest = $2;
	  }
	  
	SWITCH: for($type) {
	    /scriptalias/ && do { 
	      $scriptalias[$scount]{from} = $from; 
	      $scriptalias[$scount]{dest} = $dest;
	      $scount++;
	      last SWITCH;
	    };
	    /alias/ && do {
	      $alias[$acount]{from} = $from; 
	      $alias[$acount]{dest} = $dest;
	      $acount++;
	      last SWITCH;
	    };
	    /redirect/ && do {
	      $redirect[$rcount]{from} = $from; 
	      $redirect[$rcount]{dest} = $dest;
	      $rcount++;
	      last SWITCH;
	    };
	  }
	} elsif (/^pathcheck/i) {  # handle pathcheck lines. (look for index filenames)
	  if (/(index-names=\")(\S+)(\")/i) {
	    $indexfn = $2;
	    $indexfn =~ s/,/ /g;
	  } elsif (/require-auth/) {
	    $obj_struct{$obj_name}{require_auth} = 1;
	    if (/(realm=\")([\w|\s]+)(\")/i) {
	      $realm = $2;
	      $obj_struct{$obj_name}{realm} = $realm;
	    }
	    if (/(auth-user=\")(.+)/i) {
	      $auth_user = $2;
	      $auth_user =~ s/\".*//g;
	      $auth_user =~ s/\(|\)//g;
	      $auth_user =~ s/\|/ /g;
	      $obj_struct{$obj_name}{authuser} = $auth_user;
	    }
	  }
	} elsif (/^authtrans/i) {
	  if (/fn=\"basic-ncsa\"/i) {
	    if (/(dbm=\")([\w|\W]+\s)/i) {
	      $dbm_file = $2;
	      $dbm_file =~ s/\".*|\s//g;
	      $obj_struct{$obj_name}{dbm_file} = $dbm_file;
	    }
	    
	  }
	} elsif (/^objecttype/i) {  # look to see if server parsed html should be turned on.
	  if (/(fn=\")(shtml-hacktype)(\")/i) {
	    $serv{servparse} = 1;
	  }
	}
      } 
    } elsif (/(<object\s)(\w+=\")(.+)(\")>/i ) {   # Is this the beginning of an <object></object> block.
      $in_obj = 1;
      $obj_type = $2;
      $obj_name = $3;
      $obj_name =~ s/\*//g;
      $obj_type =~ s/=\"//;
      
      if ($obj_type =~ m/ppath/i) {
	$obj_struct{$obj_name}{ppath} = 1;
      }
    }
  }
  close (OBJ_CONF);

  $obj_struct{docroot} = $docroot;
  $obj_struct{indexfn} = $indexfn;

  return(1);
}

sub write_httpdconf {
  my $fn = shift;

  print "[Apache] Writing httpd configuration file: $fn...\n";
  open(OUT, ">$fn") || die "Couldn't open $fn for output.\n";
  print OUT qq|
# This is the main server configuration file. See URL http://www.apache.org/
# for instructions.

# Do NOT simply read the instructions in here without understanding
# what they do, if you are unsure consult the online docs. You have been
# warned.  

# Originally by Rob McCool

# ServerType is either inetd, or standalone.

ServerType standalone

# If you are running from inetd, go to "ServerAdmin".

# Port: The port the standalone listens to. For ports < 1023, you will
# need httpd to be run as root initially.

Port 80

# HostnameLookups: Log the names of clients or just their IP numbers
#   e.g.   www.apache.org (on) or 204.62.129.132 (off)

HostnameLookups $serv{dns}

# If you wish httpd to run as a different user or group, you must run
# httpd as root initially and it will switch.  

# User/Group: The name (or #number) of the user/group to run httpd as.
#  On SCO (ODT 3) use User nouser and Group nogroup
User $serv{user}
Group #-1

# ServerAdmin: Your address, where problems with the server should be
# e-mailed.

ServerAdmin you\@your.address

# ServerRoot: The directory the server's config, error, and log files
# are kept in

ServerRoot /usr/local/etc/httpd

# BindAddress: You can support virtual hosts with this option. This option
# is used to tell the server which IP address to listen to. It can either
# contain "*", an IP address, or a fully qualified Internet domain name.
# See also the VirtualHost directive.

BindAddress $serv{address}
|;
  if ($LOG_TO_OLD) {
    print OUT qq|

# ErrorLog: The location of the error log file. If this does not start
# with /, ServerRoot is prepended to it.

ErrorLog $serv{errorlog}\n

# TransferLog: The location of the transfer log file. If this does not
# start with /, ServerRoot is prepended to it.

TransferLog $serv{transferlog}

# PidFile: The file the server should log its pid to
PidFile $serv{pidlog}

|;
  } else {
    print OUT qq|

# ErrorLog: The location of the error log file. If this does not start
# with /, ServerRoot is prepended to it.

ErrorLog logs/error_log

# TransferLog: The location of the transfer log file. If this does not
# start with /, ServerRoot is prepended to it.

TransferLog logs/access_log

# PidFile: The file the server should log its pid to
PidFile logs/httpd.pid

|;
  }
  print OUT qq|
# ScoreBoardFile: File used to store internal server process information
ScoreBoardFile logs/apache_status

# ServerName allows you to set a host name which is sent back to clients for
# your server if it's different than the one the program would get (i.e. use
# "www" instead of the host's real name).
#
# Note: You cannot just invent host names and hope they work. The name you 
# define here must be a valid DNS name for your host. If you don't understand
# this, ask your network administrator.

#ServerName new.host.name

# CacheNegotiatedDocs: By default, Apache sends Pragma: no-cache with each
# document that was negotiated on the basis of content. This asks proxy
# servers not to cache the document. Uncommenting the following line disables
# this behavior, and proxies will be allowed to cache the documents.

#CacheNegotiatedDocs

# Timeout: The number of seconds before receives and sends time out
#  n.b. the compiled default is 1200 (20 minutes !)

Timeout 400

# KeepAlive: The number of Keep-Alive persistent requests to accept
# per connection. Set to 0 to deactivate Keep-Alive support

KeepAlive 5

# KeepAliveTimeout: Number of seconds to wait for the next request

KeepAliveTimeout 15

# Server-pool size regulation.  Rather than making you guess how many
# server processes you need, Apache dynamically adapts to the load it
# sees --- that is, it tries to maintain enough server processes to
# handle the current load, plus a few spare servers to handle transient
# load spikes (e.g., multiple simultaneous requests from a single
# Netscape browser).

# It does this by periodically checking how many servers are waiting
# for a request.  If there are fewer than MinSpareServers, it creates
# a new spare.  If there are more than MaxSpareServers, some of the
# spares die off.  These values are probably OK for most sites ---

MinSpareServers 5
MaxSpareServers 10

# Number of servers to start --- should be a reasonable ballpark figure.

StartServers $serv{minprocs}

# Limit on total number of servers running, i.e., limit on the number
# of clients who can simultaneously connect --- if this limit is ever
# reached, clients will be LOCKED OUT, so it should NOT BE SET TOO LOW.
# It is intended mainly as a brake to keep a runaway server from taking
# Unix with it as it spirals down...

MaxClients 150

# MaxRequestsPerChild: the number of requests each child process is
#  allowed to process before the child dies.
#  The child will exit so as to avoid problems after prolonged use when
#  Apache (and maybe the libraries it uses) leak.  On most systems, this
#  isn't really needed, but a few (such as Solaris) do have notable leaks
#  in the libraries.

MaxRequestsPerChild 30

# Proxy Server directives. Uncomment the following line to
# enable the proxy server:

#ProxyRequests On

# To enable the cache as well, edit and uncomment the following lines:

#CacheRoot /usr/local/etc/httpd/proxy
#CacheSize 5
#CacheGcInterval 4
#CacheMaxExpire 24
#CacheLastModifiedFactor 0.1
#CacheDefaultExpire 1
#NoCache adomain.com anotherdomain.edu joes.garage.com

# Listen: Allows you to bind Apache to specific IP addresses and/or
# ports, in addition to the default. See also the VirtualHost command

#Listen 3000
#Listen 12.34.56.78:80

# VirtualHost: Allows the daemon to respond to requests for more than one
# server address, if your server machine is configured to accept IP packets
# for multiple addresses. This can be accomplished with the ifconfig 
# alias flag, or through kernel patches like VIF.

# Any httpd.conf or srm.conf directive may go into a VirtualHost command.
# See alto the BindAddress entry.
 
#<VirtualHost host.foo.com>
#ServerAdmin webmaster\@host.foo.com
#DocumentRoot /www/docs/host.foo.com
#ServerName host.foo.com
#ErrorLog logs/host.foo.com-error_log
#TransferLog logs/host.foo.com-access_log
#</VirtualHost>
|;
  close(OUT);
  return(1);
}

sub write_srmconf {
  my $fn = shift;
  my $curr_alias;

  print "[Apache] Writing srm configuration file: $fn...\n";
  open(OUT, ">$fn") || die "Couldn't open $fn for output.\n";
  print OUT qq|
# With this document, you define the name space that users see of your http
# server.  This file also defines server settings which affect how requests are
# serviced, and how results should be formatted. 
  
# See the tutorials at http://www.apache.org/ for
# more information.

# Originally by Rob McCool; Adapted for Apache


# DocumentRoot: The directory out of which you will serve your
# documents. By default, all requests are taken from this directory, but
# symbolic links and aliases may be used to point to other locations.

DocumentRoot $obj_struct{docroot}

# UserDir: The name of the directory which is appended onto a user's home
# directory if a ~user request is recieved.

UserDir public_html

# DirectoryIndex: Name of the file or files to use as a pre-written HTML
# directory index.  Separate multiple entries with spaces.

DirectoryIndex $obj_struct{indexfn}

# FancyIndexing is whether you want fancy directory indexing or standard

FancyIndexing on

# AddIcon tells the server which icon to show for different files or filename
# extensions

AddIconByEncoding (CMP,/icons/compressed.gif) x-compress x-gzip

AddIconByType (TXT,/icons/text.gif) text/*
AddIconByType (IMG,/icons/image2.gif) image/*
AddIconByType (SND,/icons/sound2.gif) audio/*
AddIconByType (VID,/icons/movie.gif) video/*

AddIcon /icons/binary.gif .bin .exe
AddIcon /icons/binhex.gif .hqx
AddIcon /icons/tar.gif .tar
AddIcon /icons/world2.gif .wrl .wrl.gz .vrml .vrm .iv
AddIcon /icons/compressed.gif .Z .z .tgz .gz .zip
AddIcon /icons/a.gif .ps .ai .eps
AddIcon /icons/layout.gif .html .shtml .htm .pdf
AddIcon /icons/text.gif .txt
AddIcon /icons/c.gif .c
AddIcon /icons/p.gif .pl .py
AddIcon /icons/f.gif .for
AddIcon /icons/dvi.gif .dvi
AddIcon /icons/uuencoded.gif .uu
AddIcon /icons/script.gif .conf .sh .shar .csh .ksh .tcl
AddIcon /icons/tex.gif .tex
AddIcon /icons/bomb.gif core

AddIcon /icons/back.gif ..
AddIcon /icons/hand.right.gif README
AddIcon /icons/folder.gif ^^DIRECTORY^^
AddIcon /icons/blank.gif ^^BLANKICON^^

# DefaultIcon is which icon to show for files which do not have an icon
# explicitly set.

DefaultIcon /icons/unknown.gif

# AddDescription allows you to place a short description after a file in
# server-generated indexes.
# Format: AddDescription "description" filename

# ReadmeName is the name of the README file the server will look for by
# default. Format: ReadmeName name
#
# The server will first look for name.html, include it if found, and it will
# then look for name and include it as plaintext if found.
#
# HeaderName is the name of a file which should be prepended to
# directory indexes. 

ReadmeName README
HeaderName HEADER

# IndexIgnore is a set of filenames which directory indexing should ignore
# Format: IndexIgnore name1 name2...

IndexIgnore */.??* *~ *# */HEADER* */README* */RCS

# AccessFileName: The name of the file to look for in each directory
# for access control information.

AccessFileName .htaccess

# DefaultType is the default MIME type for documents which the server
# cannot find the type of from filename extensions.

DefaultType text/plain

# AddEncoding allows you to have certain browsers (Mosaic/X 2.1+) uncompress
# information on the fly. Note: Not all browsers support this.

AddEncoding x-compress Z
AddEncoding x-gzip gz

# AddLanguage allows you to specify the language of a document. You can
# then use content negotiation to give a browser a file in a language
# it can understand.  Note that the suffix does not have to be the same
# as the language keyword --- those with documents in Polish (whose
# net-standard language code is pl) may wish to use "AddLanguage pl .po" 
# to avoid the ambiguity with the common suffix for perl scripts.

AddLanguage en .en
AddLanguage fr .fr
AddLanguage de .de
AddLanguage da .da
AddLanguage el .el
AddLanguage it .it

# LanguagePriority allows you to give precedence to some languages
# in case of a tie during content negotiation.
# Just list the languages in decreasing order of preference.

LanguagePriority en fr de

# Redirect allows you to tell clients about documents which used to exist in
# your server's namespace, but do not anymore. This allows you to tell the
# clients where to look for the relocated document.
# Format: Redirect fakename url

|;

  for $curr_alias (1 .. $#redirect) {
    print OUT "Redirect $redirect[$curr_alias]{from}
$redirect[$curr_alias]{dest}\n";
  }

  print OUT qq|
# ScriptAlias: This controls which directories contain server scripts.
# Format: ScriptAlias fakename realname

#ScriptAlias /cgi-bin/ /usr/local/etc/httpd/cgi-bin/

|;

  foreach $curr_alias (1 .. $#scriptalias) {
    print OUT "ScriptAlias $scriptalias[$curr_alias]{from}
$scriptalias[$curr_alias]{dest}\n";
  }

  print OUT qq|
# Aliases: Add here as many aliases as you need (with no limit). The format is 
# Alias fakename realname
#Alias /icons/ /usr/local/etc/httpd/icons/

|;

  for $curr_alias (1 .. $#alias) {
    print OUT "Alias $alias[$curr_alias]{from} $alias[$curr_alias]{dest}\n";
  }

  print OUT qq|
# If you want to use server side includes, or CGI outside
# ScriptAliased directories, uncomment the following lines.

# AddType allows you to tweak mime.types without actually editing it, or to
# make certain files to be certain types.
# Format: AddType type/subtype ext1

# AddHandler allows you to map certain file extensions to "handlers",
# actions unrelated to filetype. These can be either built into the server
# or added with the Action command (see below)
# Format: AddHandler action-name ext1

# To use CGI scripts:
#AddHandler cgi-script .cgi

# To use server-parsed HTML files
|;
  if ($serv{servparse}) {
    print OUT qq|
AddType text/html .shtml
AddHandler server-parsed .shtml
|;
  } else {
     print OUT qq|
#AddType text/html .shtml
#AddHandler server-parsed .shtml
|;
  }
  print OUT qq|

# Uncomment the following line to enable Apache's send-asis HTTP file
# feature
#AddHandler send-as-is asis

# If you wish to use server-parsed imagemap files, use
AddHandler imap-file map

# To enable type maps, you might want to use
AddHandler type-map var

# Action lets you define media types that will execute a script whenever
# a matching file is called. This eliminates the need for repeated URL
# pathnames for oft-used CGI file processors.
# Format: Action media/type /cgi-script/location
# Format: Action handler-name /cgi-script/location

# For example to add a footer (footer.html in your document root) to
# files with extension .foot (e.g. foo.html.foot), you could use:
#AddHandler foot-action foot
#Action foot-action /cgi-bin/footer

# Or to do this for all HTML files, for example, use:
#Action text/html /cgi-bin/footer

# MetaDir: specifies the name of the directory in which Apache can find
# meta information files. These files contain additional HTTP headers
# to include when sending the document

#MetaDir .web

# MetaSuffix: specifies the file name suffix for the file containing the
# meta information.

#MetaSuffix .meta

# Customizable error response (Apache style)
#  these come in three flavors
#
#    1) plain text
#ErrorDocument 500 "The server made a boo boo.
#  n.b.  the (") marks it as text, it does not get output
#
#    2) local redirects
#ErrorDocument 404 /missing.html
#  to redirect to local url /missing.html
#ErrorDocument 404 /cgi-bin/missing_handler.pl
#  n.b. can redirect to a script or a document using server-side-includes.
#
#    3) external redirects
#ErrorDocument 402 http://other.server.com/subscription_info.html
#
|;
  close(OUT);
  return(1);
}

sub write_accessconf {
  my $fn = shift;

  print "[Apache] Writing access configuration file: $fn...\n";
  open(OUT, ">$fn") || die "Couldn't open $fn for output.\n";
  print OUT qq|
# access.conf: Global access configuration
# Online docs at http://www.apache.org/

# This file defines server settings which affect which types of services
# are allowed, and in what circumstances. 

# Each directory to which Apache has access, can be configured with respect
# to which services and features are allowed and/or disabled in that
# directory (and its subdirectories). 

# Originally by Rob McCool

# /usr/local/etc/httpd/ should be changed to whatever you set ServerRoot to.
#<Directory /usr/local/etc/httpd/cgi-bin>
#Options Indexes FollowSymLinks
#</Directory>

# This should be changed to whatever you set DocumentRoot to.

<Directory $serv{docroot}>

# This may also be "None", "All", or any combination of "Indexes",
# "Includes", "FollowSymLinks", "ExecCGI", or "MultiViews".

# Note that "MultiViews" must be named *explicitly* --- "Options All"
# doesn't give it to you (or at least, not yet).

Options Indexes FollowSymLinks

# This option allows you to turn on the XBitHack behavior, which allows you
# to make text/html server-parsed by activating the owner x bit with chmod. 
# This directive may be used wherever Options may, and has three
# possible arguments: Off, On or Full. If set to full, Apache will also
# add a Last-Modified header to the document if the group x bit is set.

# Unless the server has been compiled with -DXBITHACK, this function is
# off by default. To use, uncomment the following line:

#XBitHack Full

# This controls which options the .htaccess files in directories can
# override. Can also be "None", or any combination of "Options", "FileInfo", 
# "AuthConfig", and "Limit"

AllowOverride All

# Controls who can get stuff from this server.

<Limit GET>
order allow,deny
allow from all
</Limit>
</Directory>

# Allow server status reports, with the URL of http://servername/status
# Change the ".nowhere.com" to match your domain to enable.

<Location /status>
SetHandler server-status

<Limit GET>
order deny,allow
deny from all
allow from .nowhere.com
</Limit>
</Location>

# You may place any other directories or locations you wish to have
# access information for after this one.
|;
  for (keys %obj_struct) {
    next if (/default/i);
    next if (/cgi/i);
    next unless ($obj_struct{$_}{require_auth});
    print OUT qq|
<Directory $_>
AuthName $obj_struct{$_}{realm}
AuthDBMUserFile $obj_struct{$_}{dbm_file}
<Limit GET POST>
require user $obj_struct{$_}{authuser}
</Limit>
</Directory>
|;
  }
  close(OUT);
  return(1);
}

&read_magnus($MAGNUS_CONF);
&read_objs($OBJ_CONF);
&write_httpdconf($HTTPD_CONF);
&write_srmconf($SRM_CONF);
&write_accessconf($ACCESS_CONF);
exit;
if any further changes are needed rules and incentives can be enforced with this consensus 

