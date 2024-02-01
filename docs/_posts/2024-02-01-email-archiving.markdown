---
layout: default
title: "Archiving email"
---

# Records management #

The University of Washington has [records management](https://finance.uw.edu/recmgt/home) rules. A record can take many forms; any electronic or physical document can be a record.

Records can be [transitory or substantive](https://finance.uw.edu/recmgt/transitory-substantive). Ideally, people file substantive emails as they go.

Part of [records management off-boarding](https://finance.uw.edu/recmgt/Offboarding) is ensuring that the unit you're leaving has a copy of substantive records.

In my case, I've had an "Archive" folder where I put all my emails. If I wanted to review my emails for what was substantive, I'd need to read every one.

# Creating an archive #

Here's how I ended up limiting the emails I retained but ensuring that any email that's potentially substantive is kept.

## Technical setup ##

I use `offlineimap` and `notmuch`. This lets me use notmuch searches, which are way, way, way faster and more powerful than any Outlook/Exchange search. I also use a script `maildir-notmuch-sync` that synchronizes notmuch tags to the physical IMAP directories.

## Identifying logic for redundant emails ##

I used a few heuristics to remove emails that I knew would be transitory.

### Checking based on sender ###

I ended up doing a search for all the email addresses that I have sent or received mails from, using output from:

    notmuch address --deduplicate=address

I didn't want to look through every message, but I was comfortable looking through this list of addresses to identify people that I _never_ would have had a substantive record in email. I then knew it would be OK to delete the emails that showed up when searching for these addresses.

### Checking based on attachments ###

I also knew that any emails that had a `pdf`, `docx`, or `xlsx` file type would be redundant, because my process has been for substantive files to always be stored elsewhere. So, I could remove any emails with these type of attachments.

### Checking based on message type ###

I knew that any meeting invitations would not be substantive, because meeting agendas and minutes have always kept in another location.

### Checking based on subject ###

Certain subjects were easy to filter on, e.g. `^Autoreply`. Notmuch lets you do regular expression searches, e.g. with:

    notmuch search 'subject:"/^Autoreply/"'

You have to quote the search term in case there's a space in it. If the search term begins with `/` then it's considered a regular expression.

## Cleaning up duplicate files ##

In the course of this work, I learned that several physical files can correspond to one message-ID. `fdupes` can help a bit. However, the files themselves are not always exactly identical.

To clear this up, I ended up doing a number of searches using `notmuch search --output=files <query>` and then operating on the output files.

## Creating the archives ##

I wanted to create `.mbox` archives, because mbox is an open standard. To do this, I used [maildir2mbox.py](https://gist.githubusercontent.com/nyergler/1709069/raw/ed50cc60ae6fd7e6c5a3f49d308951ef183c7814/maildir2mbox.py) as a basis. I changed a couple of things:

  1. I had the program update the UNIX from date:
  
                def update_date(msg):
                   for header in ("Date", "Received"):
                      value = msg.get(header)
                      if value is None:
                         continue
                      try:
                         # 2024-01-25 jhb - Adding the UNIX from
                         email_date_str = value.split('\n')[-1].split(';')[-1].strip()
                         email_date = parsedate_tz(email_date_str)
                         out_email_date_str = time.strftime("%a %b %d %H:%M:%S %Y",
                                                            email_date[0:9])
                         from_str = "From MAILER-DAEMON %s" % (out_email_date_str)
                         msg.set_unixfrom(from_str)
                      except:
                         pass

  2. I sorted the mbox messages based on date (see also [this stackoverflow answer from user3850](https://stackoverflow.com/a/368067):
  
  
                def extract_date(email):
                    date = email.get('Date')
                    return parsedate_tz(date)
        
                # ...
                   sorted_mails = sorted(mbox, key=extract_date)
                   mbox.update(enumerate(sorted_mails))

  3. I zipped the files and labeled them with their potential destruction dates, e.g. "keep until 2025-01-01".
