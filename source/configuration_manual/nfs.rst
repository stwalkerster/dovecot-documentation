.. _nfs:

###
NFS
###

Dovecot is commonly used with NFS. However, Dovecot does **not** support
accessing the same user simultaneously by different servers. That will
result in more or less severe mailbox corruption. Note that this applies
to all mailbox access, including mail delivery.

* Use :ref:`Dovecot director <dovecot_director>` for clustering.
* Use :ref:`lmtp_server` for mail deliveries.
* Set :ref:`setting-mmap_disable` = ``yes``
* Set :ref:`setting-mail_fsync` = ``always``
* Do **not** set :ref:`setting-mail_nfs_index` or
  :ref:`setting-mail_nfs_storage` (i.e. keep them as ``no``)
* Do **not** use the ``quota-status`` service.
* Unmounted NFS mount point directory should not be writable to Dovecot
  mail processes (i.e. often the ``vmail`` user). Otherwise if the NFS
  isn't mounted for some reason and user access mails, a new empty user
  mail directory is created, which breaks things.

NFS mount options
=================

* ``nordirplus``: Disable readdirplus operations, which aren't needed by
  Dovecot. They can also slow down some NFS servers.

* ``root_squash``: Dovecot doesn't care about this. Typically Dovecot doesn't
  store any root-owned files in NFS.

* ``nolock``: This is possible to use either if ``lock_method=dotlock`` is
  used, or as a slightly unsafe optimization (see below).

Optimizations
=============

Potential optimizations to use:

* mdbox format is likely more efficient to use than the sdbox format. The
  downside is that it requires running periodic ``doveadm purge`` for each
  user. This commands should be run via the doveadm directors so they are run
  in the proper backends.
* Use ``mail_location = ...:VOLATILEDIR=/dev/shm/dovecot/%2.256Nu/%u`` to
  store some temporary files (e.g. lock files) in tmpfs rather than NFS.
* Use ``mail_location = ...:LISTINDEX=/dev/shm/dovecot/%2.256Nu/%u/dovecot.list.index``
  to store mailbox list indexes in tmpfs rather than NFS. This makes moving
  the users between backends more expensive though. It also needs a way to
  delete the list indexes for users that have already moved to different
  backends.
* Use ``mail_location = ...:INDEX=/fast/%2.256Nu/%u:ITERINDEX`` to use
  "smaller fast storage" for index files and "larger slow storage" for mail
  files. The ``ITERINDEX`` is used to list mailboxes via the fast index
  storage rather than the slow mail storage.
* Use ``mail_location = ...:ALT=/slow/%2.256Nu/%u:NOALTCHECK`` to use
  "smaller fast storage" for new mails and "larger slow storage" for old
  mails. The ``doveadm altmove`` command needs to be run periodically. The
  ``NOALTCHECK`` disables a sanity check to make sure alt storage path doesn't
  unexpectedly change.
* Slightly unsafe: Use ``nolock`` mount option to keep all locking within the
  server. Assuming director works perfectly, there is no need to use locking
  across NFS. Each user only locks their own files, and the user should only
  be accessed by a single server at a time. Unfortunately, this doesn't work
  100% of the time so in some rare situations the same user can become
  accessed by multiple servers simultaneously. In those situations the mails
  are more likely to become corrupted if ``nolock`` is used.

Clock synchronization
=====================

Run ntpd in the NFS server and all the NFS clients to make sure their
clocks are synchronized. They should always be less than 1 second apart
from each other.

Clustering without director
===========================

Some people are using Dovecot in a cluster without the director service.
Below are some suggestions for improving the reliability of this
configuration.

.. warning:: This method is almost guaranteed to give random errors and can
             potentially lose emails.

* Configure load balancer so that connections from the same source IP are
  redirected to the same Dovecot server. This way a single client using
  multiple IMAP connections doesn't immediately cause problems.

* Set :ref:`setting-mail_nfs_index` = ``yes`` and
  :ref:`setting-mail_nfs_storage` = ``yes``. These will attempt to flush the NFS
  caches at appropriate times. However, it doesn't work perfectly.

    * Disabling NFS attribute cache helps a lot in getting rid of caching
      related errors, but this makes performance MUCH worse and increases
      the load on NFS server. This can usually be done by giving ``actimeo=0``
      or ``noac`` mount option.

* Make sure NFS lockd works properly. If it doesn't, use
  :ref:`setting-lock_method` = ``dotlock``. However, this degrades performance.

* Use Maildir mailbox format instead of sdbox/mdbox. Maildir is much more
  resistant to corruption.

    * Deliver mails in a way that it doesn't update Dovecot index files.
      Either don't use Dovecot LDA/LMTP, or configure it to use in-memory
      index files::

          protocol lda {
            mail_location = maildir:~/Maildir:INDEX=MEMORY
          }

