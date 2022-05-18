---
layout: default
title: "Checking Office365 email with Emacs"
---
My UW email is served via Office365. Outlook is a fine email client, but as part of using Emacs org-mode, I wanted the ability to check my email via Emacs.

Why do this?

- This setup allows me to relate my org-mode tasks to email.
- I can use org-mode to write and format my messages, which sometimes enables me to do things that I can't easily do via Outlook.
- I can use `yasnippet` to create email response templates.
- I can much more easily reply in-line to emails.
- I can use notmuch filters to process my email and search archives.
- I can use Emacs keybindings to write emails, e.g. `C-c C-e` elides a set of lines in email and `C-c C-f C-s` lets me change the subject in a reply.

This post describes how I set this up, in case it's useful for others.

The moving parts for this setup are:

- `authinfo.gpg` - file that Emacs and offlineimap can both use to store/retrieve passwords
- [offlineimap](http://www.offlineimap.org/) - transfers email to/from Office365
- [maildir-notmuch-sync](https://raw.githubusercontent.com/altercation/es-bin-arch/master/maildir-notmuch-sync) - enables moving messages between folders on the filesystem, which enables moving messages in Office365 folders too
- [notmuch](https://notmuchmail.org/) - enables Emacs access + search for email

Also as context, I'm doing this on macOS and I use [macports](https://www.macports.org/).

# macports

I installed these ports:

    sudo port install notmuch offlineimap gnupg2 curl

`gnupg2` is used to read/write from `~/.authinfo.gpg`.

`curl` is needed to ensure the CA certificate bundle has been downloaded, which is referenced in `.offlineimaprc` below.

# offlineimap

## OAuth

offlineimap supports OAuth authentication (instead of username + password authentication).

Please see <https://github.com/UvA-FNWI/M365-IMAP>, which is a great guide. Briefly, to get OAuth credentials for Office365, you need to use [the Azure portal (portal.azure.com)](https://portal.azure.com). You need to use Azure active directory, go to "App registrations," and then create a new application per this guide.

However, the `oauth2_request_url` in the guide, `https://login.microsoftonline.com/common/oauth2/v2.0/token`, is not necessarily correct. You may need to use a more specific endpoint than `common`, something along the lines of 
`https://login.microsoftonline.com/FIXME/oauth2/v2.0/token`. You will need to replace `FIXME` with what is called the "tenant ID." You can find the tenant ID from going to Azure Active Directory, navigating up the breadcrumbs to your organization, and then clicking "Overview."

Once you have your OAuth2 information, when following the below steps:

- populate `oauth2_request_url` in `.offlineimaprc`
- populate `oauth2_client_id` in `.offlineimaprc`
- populate `OAuth2ClientSecret` in `.authinfo.gpg`
- populate `OAuth2RefreshToken` in `.authinfo.gpg`

## .offlineimap.py

This script will be parsed by `.offlineimaprc` to define the function `get_password_emacs`, which will use GPG to open `~/.authinfo.gpg`.

{% highlight python %}
#!/usr/bin/python
import re, os

def get_password_emacs(machine, login, port):
    s = "machine %s login %s port %s password ([^ ]*)\n" % (machine, login, port)
    p = re.compile(s)
    authinfo = os.popen("gpg -q --no-tty -d ~/.authinfo.gpg").read()
    return p.search(authinfo).group(1)
{% endhighlight %}

## .offlineimaprc

Copy this to `~/.offlineimaprc` and change everywhere it says `FIXME`. This will put a copy of your Office365 email into `~/Mail`.

{% highlight conf %}
[general]
accounts = Office365
pythonfile = ~/.offlineimap.py

[Account Office365]
localrepository = Local
remoterepository = Remote
presynchook = ~/bin/maildir-notmuch-sync "/Users/FIXME/Mail"
postsynchook = ~/bin/maildir-notmuch-sync "/Users/FIXME/Mail"

[Repository Local]
type = Maildir
localfolders = ~/Mail
nametrans = lambda folder: {'sent': 'Sent Items',
                            'deleted': 'Deleted Items',
                           }.get(folder, folder)
folderfilter = lambda folder: folder not in [
    'drafts',
   ]

[Repository Remote]
type = IMAP
sslcacertfile = /opt/local/share/curl/curl-ca-bundle.crt
remotehost = outlook.office365.com
remoteuser = FIXME@uw.edu
auth_mechanisms = XOAUTH2
oauth2_request_url = FIXME
oauth2_client_id = FIXME
oauth2_client_secret_eval = get_password_emacs("outlook.office365.com", "OAuth2ClientSecret", "443")
oauth2_refresh_token_eval = get_password_emacs("outlook.office365.com", "OAuth2RefreshToken", "443")
folderfilter = lambda folder: folder not in [
  'Calendar',
  'Calendar/Birthdays',
  'Calendar/United States holidays',
  'Clutter',
  'Contacts',
  'Contacts/Skype for Business Contacts',
  'Conversation History',
  'Deleted Items',
  'Drafts',
  'Journal',
  'Junk Email',
  'Notes',
  'Outbox',
  'RSS Feeds',
  'Sync Issues',
  'Sync Issues/Conflicts',
  'Sync Issues/Local Failures',
  'Sync Issues/Server Failures',
  'Tasks',
  ]
maxconnections = 1
singlethreadperfolder = yes
nametrans = lambda folder: {'Sent Items': 'sent',
                            'Deleted Items': 'deleted',
                           }.get(folder, folder)
{% endhighlight %}

# authinfo.gpg

If you don't already have a GPG public/private key pair, you need to [create a public and private key](https://www.redhat.com/sysadmin/creating-gpg-keypairs).

You can then tell Emacs to visit `~/.authinfo.gpg` and it ought to prompt you to encrypt the file when you save via [EasyPG Assistant aka epa](https://www.gnu.org/software/emacs/manual/html_node/epa/Encrypting_002fdecrypting-gpg-files.html).

offlineimap and/or Emacs need three passwords:

    machine outlook.office365.com login OAuth2ClientSecret port 443 password FIXME
    machine outlook.office365.com login OAuth2RefreshToken port 443 password FIXME
    machine smtp.uw.edu login FIXME port 465 password FIXME

The outlook.office365.com credentials are used by offlineimap. The smtp.uw.edu credentials are used by Emacs when sending email.

# ~/bin/maildir-notmuch-sync

Create a directory `~/bin` if it doesn't already exist and put <https://raw.githubusercontent.com/altercation/es-bin-arch/master/maildir-notmuch-sync>
in it. This script will move files in your `~/Mail` folders as their labels change in notmuch. This enables notmuch tags to move email in your Office365 mailbox.

You'll need to make the script executable:

    chmod +x ~/bin/maildir-notmuch-sync

I recommend you edit this script. In my case, my Office365 archive folder is called `Archive`. Here's a diff of what I changed:

{% highlight diff %}
< 
---
> # 2019-02-21 jhb - This came from
> #  https://raw.githubusercontent.com/altercation/es-bin-arch/master/maildir-notmuch-sync
> # 
67,68c69,70
< SENT="sent"         # (mutt's record dir) - both the tag and folder
< ARCHIVE="archive"   # (mutt's mbox/received) - both the tag and folder
---
> SENT="Sent Items"         # (mutt's record dir) - both the tag and folder
> ARCHIVE="Archive"   # (mutt's mbox/received) - both the tag and folder
330c332,333
< NOTMUCH_ROOT="${NOTMUCH_ROOT%/}"
---
> # 2019-02-21 jhb - added a slash back
> NOTMUCH_ROOT="${NOTMUCH_ROOT%/}/"
339a343,348
> if [ "$#" -lt 1 ]
> then
>     echo "Pass the ACCOUNT ROOT as the first argument!"
>     exit 1
> fi
> 
356c365
< MAILBOXES_FULL_PATHS="$(echo "$(find $MAILDIR_ACCOUNT_ROOT -name "cur" -type d -exec dirname '{}' \;)" | sort;)"
---
> MAILBOXES_FULL_PATHS="$(echo "$(find "$MAILDIR_ACCOUNT_ROOT" -name "cur" -type d -exec dirname '{}' \;)" | sort;)"
455d463
<         AND tag:"$ARCHIVE" \
486c494
<             if $RUNCMD "cp \"$THIS_MESSAGE_SOURCE_PATH\" \"$THIS_MAILDIR_FULL_PATH/cur\""; then
---
>             if $RUNCMD "mv \"$THIS_MESSAGE_SOURCE_PATH\" \"$THIS_MAILDIR_FULL_PATH/cur\""; then
531d538
<         AND tag:"$ARCHIVE" \
671a679
> IFS=$'\n'
674c682,683
<     Notmuch_State_To_Maildir__Remove_From_Maildir $MAILBOX_FULL_PATH
---
>     # 2019-04-19: removing this and using mv instead of cp above
> #    Notmuch_State_To_Maildir__Remove_From_Maildir $MAILBOX_FULL_PATH
{% endhighlight %}

# Checkpoint: getting your email

At this point, you should be able to run `offlineimap` to check your email. You may need to initialize notmuch first, though:

    notmuch setup

My notmuch configuration looks like this:

    built_with.compact=true
    built_with.field_processor=true
    built_with.retry_lock=true
    built_with.sexp_queries=false
    database.autocommit=8000
    database.backup_dir=/Users/FIXME/Mail/.notmuch/backups
    database.hook_dir=/Users/FIXME/Mail/.notmuch/hooks
    database.mail_root=/Users/FIXME/Mail
    database.path=/Users/FIXME/Mail
    maildir.synchronize_flags=true
    new.ignore=
    new.tags=new;
    search.exclude_tags=deleted;spam;
    user.name=FIXME
    user.primary_email=FIXME@uw.edu

You can see your config with `notmuch config list`. You can then change the configuration options to match the above with `notmuch config set`.

Once everything is setup, you should be able to run

    offlineimap

This will likely take a long time.

If you have issues, you can comment out the `presynchook` and `postsynchook` statements in `~/.offlineimaprc` until offlineimap itself is working.

# Adding notmuch

If you had to comment out the sync hooks, after you've downloaded your email you should run `~/bin/maildir-notmuch-sync`. Then you should be able to uncomment the sync hooks. These should run notmuch.

At this point you should be able to run notmuch via the command line, using something like

    notmuch search test
	
and seeing whatever emails have `test` in them.

# notmuch in Emacs

I have this in my Emacs settings to get notmuch started:

{% highlight elisp %}
(use-package notmuch
  :bind (("C-x E" . notmuch))
  )
(eval-after-load 'notmuch-show
  '(define-key notmuch-show-mode-map "`" 'notmuch-show-apply-tag-macro))

(eval-after-load 'notmuch-search
  '(define-key notmuch-search-mode-map "`" 'notmuch-search-apply-tag-macro))
  
; this enables org-mode to link to notmuch searches and messages
(use-package ol-notmuch
  :ensure nil
  )

(setq mail-user-agent 'notmuch-user-agent)

; this lets you send email. your sent message is then synced via
; offlineimap to Office365.

(setq user-mail-address "FIXME@uw.edu"
      send-mail-function    'smtpmail-send-it
      smtpmail-smtp-server  "smtp.uw.edu"
      smtpmail-smtp-user "FIXME"
      smtpmail-stream-type  'ssl
      smtpmail-smtp-service 465
      smtpmail-auth-credentials (expand-file-name "~/.authinfo.gpg")
      notmuch-poll-script "/opt/local/bin/offlineimap"
      )
{% endhighlight %}

The above binds `C-x E` to notmuch mode (globally). It also lets me type a `` ` `` to move a message to another label. Then you can define `notmuch-show-tag-macro-alist` (shortcut keys, e.g. `` `f `` to move a message to the tag "follow-up") and `notmuch-archive-tags` (tags to remove/add when archiving a message).

When I need to create a task for a message I then type `C-c c t`, which I've configured with org-capture to capture a new task with a link to the message.

# sending HTML email

The last part of this setup is being able to send HTML email. Virtually everyone is expecting to receive HTML emails.

{% highlight elisp %}
; enable org-mime, which will convert email to HTML
(use-package org-mime
  :config (setq org-mime-export-options '(:preserve-breaks t))
  (add-hook 'message-mode-hook
	    (lambda ()
	      (local-set-key "\C-c\M-o" 'org-mime-htmlize))))

; function I wrote w/ org-mime maintainer that warns you if you're sending
; an email and it's not HTML
(add-hook 'message-send-hook 'org-mime-confirm-when-no-multipart)

; change my email font:
(add-hook 'org-mime-html-hook
      (lambda ()
        (goto-char (point-min))
        (insert "<div style=\"font-family:Georgia,serif\">")
        (goto-char (point-max))
        (insert "</div>")))

; use mml-mode for the top part of the email you're drafting + org-mode
; for the bottom part
(use-package polymode
  :config
  (define-hostmode poly-mml-hostmode :mode 'notmuch-message-mode)
  (define-innermode jb-poly-org-innermode
    :mode 'org-mode
    :head-matcher "^--text follows this line--$"
    :tail-matcher "^THISNEVEREXISTS$"
    :head-mode 'host
    :tail-mode 'org-mode)
  (define-polymode poly-org-mode
    :hostmode 'poly-mml-hostmode
    :innermodes '(jb-poly-org-innermode))
  (add-hook 'mml-mode-hook
	    (lambda () (local-set-key (kbd "C-c o") #'poly-org-mode)))
  )
{% endhighlight %}

With the above, when I'm sending an email, I can type `C-c o` to have the body rendered as org-mode by Emacs. Then, when I'm ready to send the message, I go to the top of the buffer where mml-mode is still running (e.g. with `C-<`), and then I hit `C-c M-o` to call org-mime to render the body as two multipart sections: text and HTML.

The text in the email is written in org-mode formatting, which most people won't see. The HTML version is the org-mime rendering.

If I accidentally try to send the message without doing this first, the `'org-mime-confirm-when-no-multipart` message-send-hook will ask me to confirm that I really want to send just text.

# Next steps

The above took _a long_ time to figure out. It has enabled me to do some weird things that otherwise would be impossible; most notably it has enabled me to build scripts of notmuch searches to find transitory emails and mark them for permanent deletion. I've also been able to write a reply parser that pre-processes my reply emails to remove duplicate signatures.

If you're looking to use offlineimap and/or notmuch and/or Emacs, I hope this helps you!
