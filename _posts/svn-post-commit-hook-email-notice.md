title: 通过 post-commit hook 实现 SVN Commit 邮件通知
date: 2015-03-11 20:09:03
tags:
 - svn
 - 运维
categories:
 - ITOM
 - Revision control
 - svn
---
svn 可以通过 post-commit hook 实现 commit 时发送邮件通知。
其实现原理为：
用户在提交代码至 svn 完成后触发 post-commit 钩子脚本
在 post-commit 中调用 diff 工具和 sendmail 工具发送邮件

常用的解决方案是使用 subversion-tools 里的 commit-email.pl

需要安装 perl 和其它相关工具就不说了，却啥装啥，不过最新版的 subversion-tools 安装后找不到 hook-scripts，最终找到源码，有些兼容性问题做了修改，附在下面

> 乱码问题，svn log 的乱码已经解决，但是 diff 的乱码暂时没找到解决方案，如有解求告知

<!--more-->

修改 svn/your_repos/hooks/post-commit

```shell
REPOS="$1"
REV="$2"
 
# 可以把 log 编码搞定，但是 diff 中文乱码还没辙
export LANG=zh_CN.GB2312
/home/svn/hook-scripts/commit-email.pl "$REPOS" "$REV" -s "[SVN Commit]" -h yekezhong.com yekezhong@gmail.com
```

附送修改后的 commit-email.pl，其中 CONFIGURATION SECTION 需要配置 sendmail 和 svnlook 的路径

```perl
#!/usr/bin/perl -w

# commit-email.pl: send a commit email for commit REVISION in
# repository REPOS to some email addresses.
#
# Usage: commit-email.pl REPOS REVISION [OPTIONS] [EMAIL-ADDR ...]
#
# Options:
#    -h hostname       :  Hostname to append to author for 'From:'
#    -l logfile        :  File to which mail contents should be appended
#    -r email_address  :  Set email Reply-To header to this email address
#    -s subject_prefix :  Subject line prefix
#    
# ====================================================================
# Copyright (c) 2000-2002 CollabNet.  All rights reserved.
#
# This software is licensed as described in the file COPYING, which
# you should have received as part of this distribution.  The terms
# are also available at http://subversion.tigris.org/license-1.html.
# If newer versions of this license are posted there, you may use a
# newer version instead, at your option.
#
# This software consists of voluntary contributions made by many
# individuals.  For exact contribution history, see the revision
# history and logs, available at http://subversion.tigris.org/.
# ====================================================================

use strict;
use Carp;
use Getopt::Long;

######################################################################
#  CONFIGURATION SECTION
######################################################################

# sendmail path
my $sendmail = "/usr/sbin/sendmail";

# svnlook path
my $svnlook = "/usr/bin/svnlook";

######################################################################
# Initial setup/command-line handling

# now, we see if there are any options included in the argument list
my $logfile = '';
my $hostname = '';
my $reply_to = '';
my $subject_prefix = '';

GetOptions('hostname=s' => \$hostname,
           'logfile=s'  => \$logfile,
           'reply_to=s' => \$reply_to,
           'subject=s'  => \$subject_prefix)
    or &usage;

# check that there are enough remaining command line options
&usage("$0: too few arguments") unless @ARGV > 2;

# get the REPOS from the arguments
my $repos = shift @ARGV;
my @temp = split "/", $repos;
my $repos_name = $temp[3];

# get the REVISION from the arguments
my $rev = shift @ARGV;

# initialize the EMAIL_ADDRS to the remaining arguments
my @email_addrs = @ARGV;

######################################################################
# Harvest data using svnlook

# change into /tmp so that svnlook diff can create its .svnlook directory
my $tmp_dir = '/tmp';
chdir($tmp_dir)
    or die "$0: cannot chdir `$tmp_dir': $!\n";

# get the auther, date, and log from svnlook
my @svnlooklines = &read_from_process($svnlook, 'info', '-r', $rev, $repos);
my $author = shift @svnlooklines;
my $date = shift @svnlooklines;
shift @svnlooklines;
my @log = map { "$_\n" } @svnlooklines;

# figure out what directories have changed (using svnlook)
my @dirschanged = &read_from_process($svnlook, 'dirs-changed', '-r', $rev, $repos);
my $rootchanged = 0;
grep 
{
    # lose the trailing slash if one exists (except in the case of '/')
    $rootchanged = 1 if ($_ eq '/');
    $_ =~ s/(.+)[\/\\]$/$1/;
} 
@dirschanged; 

# figure out what's changed (using svnlook)
@svnlooklines = &read_from_process($svnlook, 'changed', '-r', $rev, $repos);

# parse the changed nodes
my @adds = ();
my @dels = ();
my @mods = ();
foreach my $line (@svnlooklines)
{
    my $path = '';
    my $code = '';

    # split the line up into the modification code (ignore propmods) and path
    if ($line =~ /^(.).  (.*)$/)
    {
        $code = $1;
        $path = $2;
    }

    if ($code eq 'A') {
        push (@adds, "   $path\n");
    }
    elsif ($code eq 'D') {
        push (@dels, "   $path\n");
    }
    else {
        push (@mods, "   $path\n");
    }
}

# get the diff from svnlook
my @difflines = &read_from_process($svnlook, 'diff', '-r', $rev, $repos);

######################################################################
# Mail headers

# collapse the list of changed directories
my @commonpieces = ();
my $commondir = '';
if (($rootchanged == 0) and (scalar @commonpieces > 1))
{
    my $firstline = shift (@dirschanged);
    push (@commonpieces, split ('/', $firstline));
    foreach my $line (@dirschanged)
    {
        my @pieces = ();
        my $i = 0;
        push (@pieces, split ('/', $line));
        while (($i < scalar @pieces) and ($i < scalar @commonpieces))
        {
            if ($pieces[$i] ne $commonpieces[$i])
            {
                splice (@commonpieces, $i, (scalar @commonpieces - $i));
                last;
            }
            $i++;
        }
    }
    unshift (@dirschanged, $firstline);
    if (scalar @commonpieces)
    {
        $commondir = join ('/', @commonpieces);
        grep
        {
            s/^$commondir\/(.*)/$1/eg;
        }
        @dirschanged;
    }
}
my $dirlist = join (' ', @dirschanged);


my $userlist = join (' ', @email_addrs); 
my $subject = '';
if ($commondir ne '')
{
    $subject = "rev $rev - in $commondir: $dirlist";
}
else
{
    $subject = "$author r$rev\@$repos_name - $dirlist";
}
if ($subject_prefix =~ /\w/)
{
    $subject = "$subject_prefix $subject";
}
my $mail_from = $author;
if ($hostname =~ /\w/)
{
    $mail_from = "$mail_from\@$hostname";
}

my @output;
push (@output, "To: $userlist\n");
push (@output, "From: $mail_from\n");
push (@output, "Subject: $subject\n");
push (@output, "Reply-to: $reply_to\n") if $reply_to;
push (@output, "\n");

# mail body
push (@output, "Author: <b>$author</b>\n");
push (@output, "Date: <b>$date</b>\n");
push (@output, "Repos: <b>$repos</b>\n");
push (@output, "New Revision: <b>$rev</b>\n");
push (@output, "\n");

push (@output, "###################################\n");
push (@output, "Log:\n");
push (@output, @log);
push (@output, "\n");


push (@output, "\n###################################\n");
if (scalar @adds)
{
    @adds = sort @adds;
    push (@output, "Added:\n");
    push (@output, @adds);
}
if (scalar @dels)
{
    @dels = sort @dels;
    push (@output, "Removed:\n");
    push (@output, @dels);
}
if (scalar @mods)
{
    @mods = sort @mods;
    push (@output, "Modified:\n");
    push (@output, @mods);
}
push (@output, "\n###################################\n");
push (@output, map { "$_\n" } @difflines);


# dump output to logfile (if its name is not empty)
if ($logfile =~ /\w/)
{
    open (LOGFILE, ">> $logfile") 
        or die ("Error opening '$logfile' for append");
    print LOGFILE @output;
    close LOGFILE;
}

# open a pipe to 'sendmail'
if (($sendmail =~ /\w/) and ($userlist =~ /\w/))
{
    open (SENDMAIL, "| $sendmail $userlist") 
        or die ("Error opening a pipe to sendmail");
    print SENDMAIL @output;
    close SENDMAIL;
}

exit 0;

sub usage {
  warn "@_\n" if @_;
  die "usage: $0 [options] REPOS REVNUM email_address1 [email_address2 ... ]]\n",
      "options are\n",
      "  -h hostname        Hostname to append to author for 'From:'\n",
      "  -l logfile         File to which mail contents should be appended\n",
      "  -r email_address   Set email Reply-To header to this email address\n",
      "  -s subject_prefix  Subject line prefix\n";
}

sub safe_read_from_pipe {
  unless (@_) {
    croak "$0: safe_read_from_pipe passed no arguments.\n";
  }
  print "Running @_\n";
  my $pid = open(SAFE_READ, '-|');
  unless (defined $pid) {
    die "$0: cannot fork: $!\n";
  }
  unless ($pid) {
    open(STDERR, ">&STDOUT") or
      die "$0: cannot dup STDOUT: $!\n";
    exec(@_) or
      die "$0: cannot exec `@_': $!\n";
  }
  my @output;
  while (<SAFE_READ>) {
    chomp;
    push(@output, $_);
  }
  close(SAFE_READ);
  my $result = $?;
  my $exit   = $result >> 8;
  my $signal = $result & 127;
  my $cd     = $result & 128 ? "with core dump" : "";
  if ($signal or $cd) {
    warn "$0: pipe from `@_' failed $cd: exit=$exit signal=$signal\n";
  }
  if (wantarray) {
    return ($result, @output);
  } else {
    return $result;
  }
}

sub read_from_process {
  unless (@_) {
    croak "$0: read_from_process passed no arguments.\n";
  }
  my ($status, @output) = &safe_read_from_pipe(@_);
  if ($status) {
    return ("$0: @_ failed with this output:", @output);
  } else {
    return @output;
  }
}

```
