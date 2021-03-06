Sending Mail
============

Djrill handles **all** outgoing email sent through Django's standard
:mod:`django.core.mail` package, including :func:`~django.core.mail.send_mail`,
:func:`~django.core.mail.send_mass_mail`, the :class:`~django.core.mail.EmailMessage` class,
and even :func:`~django.core.mail.mail_admins`.

If you'd like to selectively send only some messages through Mandrill,
there is a way to :ref:`use multiple email backends <multiple-backends>`.


.. _django-send-support:

Django Email Support
--------------------

Djrill supports most of the functionality of Django's :class:`~django.core.mail.EmailMessage`
and :class:`~django.core.mail.EmailMultiAlternatives` classes.

Some notes and limitations:

**Display Names**
    All email addresses (from, to, cc, bcc) can be simple
    ("email\@example.com") or can include a display name
    ("Real Name <email\@example.com>").

**CC and BCC Recipients**
    Djrill properly identifies "cc" and "bcc" recipients to Mandrill.

    Note that you may need to set the Mandrill option :attr:`preserve_recipients`
    to `!True` if you want recipients to be able to see who else was included
    in the "to" list.


.. _sending-html:

**HTML/Alternative Parts**
    To include an HTML version of a message, use
    :meth:`~django.core.mail.EmailMultiAlternatives.attach_alternative`:

    .. code-block:: python

        from django.core.mail import EmailMultiAlternatives

        msg = EmailMultiAlternatives("Subject", "text body",
                                     "from@example.com", ["to@example.com"])
        msg.attach_alternative("<html>html body</html>", "text/html")

    Djrill allows a maximum of one
    :meth:`~django.core.mail.EmailMultiAlternatives.attach_alternative`
    on a message, and it must be ``mimetype="text/html"``.
    Otherwise, Djrill will raise :exc:`~djrill.NotSupportedByMandrillError` when you
    attempt to send the message. (Mandrill doesn't support sending multiple html
    alternative parts, or any non-html alternatives.)


.. _sending-attachments:

**Attachments**
    Djrill will send a message's attachments. (Note that Mandrill may impose limits
    on size and type of attachments.)

    Also, if an image attachment has a Content-ID header, Djrill will tell Mandrill
    to treat that as an embedded image rather than an ordinary attachment.
    (For an example, see :meth:`~DjrillBackendTests.test_embedded_images`
    in :file:`tests/test_mandrill_send.py`.)

.. _message-headers:

**Headers**
    Djrill accepts additional headers and passes them along to Mandrill:

    .. code-block:: python

        msg = EmailMessage( ...
            headers={'Reply-To': "reply@example.com", 'List-Unsubscribe': "..."}
        )

    .. note::

       Djrill also supports the `reply_to` param added to
       :class:`~django.core.mail.EmailMessage` in Django 1.8.
       (If you provide *both* a 'Reply-To' header and the `reply_to` param,
       the header will take precedence.)


.. _mandrill-send-support:

Mandrill-Specific Options
-------------------------

Most of the options from the Mandrill
`messages/send API <https://mandrillapp.com/api/docs/messages.html#method=send>`_
`message` struct can be set directly on an :class:`~django.core.mail.EmailMessage`
(or subclass) object.

.. note::

    You can set global defaults for common options with the
    :setting:`MANDRILL_SETTINGS` setting, to avoid having to
    set them on every message.


.. These attributes are in the same order as they appear in the Mandrill API docs...

.. attribute:: important

    ``Boolean``: whether Mandrill should send this message ahead of non-important ones.

.. attribute:: track_opens

    ``Boolean``: whether Mandrill should enable open-tracking for this message.
    Default from your Mandrill account settings. ::

        message.track_opens = True

.. attribute:: track_clicks

    ``Boolean``: whether Mandrill should enable click-tracking for this message.
    Default from your Mandrill account settings.

    .. note::

        Mandrill has an option to track clicks in HTML email but not plaintext, but
        it's *only* available in your Mandrill account settings. If you want to use that
        option, set it at Mandrill, and *don't* set the ``track_clicks`` attribute here.

.. attribute:: auto_text

    ``Boolean``: whether Mandrill should automatically generate a text body from the HTML.
    Default from your Mandrill account settings.

.. attribute:: auto_html

    ``Boolean``: whether Mandrill should automatically generate an HTML body from the plaintext.
    Default from your Mandrill account settings.

.. attribute:: inline_css

    ``Boolean``: whether Mandrill should inline CSS styles in the HTML.
    Default from your Mandrill account settings.

.. attribute:: url_strip_qs

    ``Boolean``: whether Mandrill should ignore any query parameters when aggregating
    URL tracking data. Default from your Mandrill account settings.

.. attribute:: preserve_recipients

    ``Boolean``: whether Mandrill should include all recipients in the "to" message header.
    Default from your Mandrill account settings.

.. attribute:: view_content_link

    ``Boolean``: set False on sensitive messages to instruct Mandrill not to log the content.

.. attribute:: tracking_domain

    ``str``: domain Mandrill should use to rewrite tracked links and host tracking pixels
    for this message. Useful if you send email from multiple domains.
    Default from your Mandrill account settings.

.. attribute:: signing_domain

    ``str``: domain Mandrill should use for DKIM signing and SPF on this message.
    Useful if you send email from multiple domains.
    Default from your Mandrill account settings.

.. attribute:: return_path_domain

    ``str``: domain Mandrill should use for the message's return-path.

.. attribute:: merge_language

    ``str``: the merge tag language if using merge tags -- e.g., "mailchimp" or "handlebars".
    Default from your Mandrill account settings.

.. attribute:: global_merge_vars

    ``dict``: merge variables to use for all recipients (most useful with :ref:`mandrill-templates`). ::

        message.global_merge_vars = {'company': "ACME", 'offer': "10% off"}

    Merge data must be strings or other JSON-serializable types.
    (See :ref:`formatting-merge-data` for details.)

.. attribute:: merge_vars

    ``dict``: per-recipient merge variables (most useful with :ref:`mandrill-templates`). The keys
    in the dict are the recipient email addresses, and the values are dicts of merge vars for
    each recipient::

        message.merge_vars = {
            'wiley@example.com': {'offer': "15% off anvils"},
            'rr@example.com':    {'offer': "instant tunnel paint"}
        }

    Merge data must be strings or other JSON-serializable types.
    (See :ref:`formatting-merge-data` for details.)

.. attribute:: tags

    ``list`` of ``str``: tags to apply to the message, for filtering reports in the Mandrill
    dashboard. (Note that Mandrill prohibits tags longer than 50 characters or starting with
    underscores.) ::

        message.tags = ["Order Confirmation", "Test Variant A"]

.. attribute:: subaccount

    ``str``: the ID of one of your subaccounts to use for sending this message.

.. attribute:: google_analytics_domains

    ``list`` of ``str``: domain names for links where Mandrill should add Google Analytics
    tracking parameters. ::

        message.google_analytics_domains = ["example.com"]

.. attribute:: google_analytics_campaign

    ``str`` or ``list`` of ``str``: the utm_campaign tracking parameter to attach to links
    when adding Google Analytics tracking. (Mandrill defaults to the message's from_email as
    the campaign name.)

.. attribute:: metadata

    ``dict``: metadata values Mandrill should store with the message for later search and
    retrieval. ::

        message.metadata = {'customer': customer.id, 'order': order.reference_number}

    Mandrill restricts metadata keys to alphanumeric characters and underscore, and
    metadata values to numbers, strings, boolean values, and None (null).

.. attribute:: recipient_metadata

    ``dict``: per-recipient metadata values. Keys are the recipient email addresses,
    and values are dicts of metadata for each recipient (similar to
    :attr:`merge_vars`)

    Mandrill restricts metadata keys to alphanumeric characters and underscore, and
    metadata values to numbers, strings, boolean values, and None (null).

.. attribute:: async

    ``Boolean``: whether Mandrill should use an async mode optimized for bulk sending.

.. attribute:: ip_pool

    ``str``: name of one of your Mandrill dedicated IP pools to use for sending this message.

.. attribute:: send_at

    `datetime` or `date` or ``str``: instructs Mandrill to delay sending this message
    until the specified time. Example::

        msg.send_at = datetime.utcnow() + timedelta(hours=1)

    Mandrill requires a UTC string in the form ``YYYY-MM-DD HH:MM:SS``.
    Djrill will convert python dates and datetimes to this form.
    (Dates will be given a time of 00:00:00.)

    .. note:: Timezones

        Mandrill assumes :attr:`!send_at` is in the UTC timezone,
        which is likely *not* the same as your local time.

        Djrill will convert timezone-*aware* datetimes to UTC for you.
        But if you format your own string, supply a date, or a
        *naive* datetime, you must make sure it is in UTC.
        See the python `datetime` docs for more information.

        For example, ``msg.send_at = datetime.now() + timedelta(hours=1)``
        will try to schedule the message for an hour from the current time,
        but *interpreted in the UTC timezone* (which isn't what you want).
        If you're more than an hour west of the prime meridian, that will
        be in the past (and the message will get sent immediately). If
        you're east of there, the message might get sent quite a bit later
        than you intended. One solution is to use `utcnow` as shown in
        the earlier example.

    .. note::

        Scheduled sending is a paid Mandrill feature. If you are using
        a free Mandrill account, :attr:`!send_at` won't work.


All the Mandrill-specific attributes listed above work with *any*
:class:`~django.core.mail.EmailMessage`-derived object, so you can use them with
many other apps that add Django mail functionality.

If you have questions about the python syntax for any of these properties,
see :class:`DjrillMandrillFeatureTests` in :file:`tests/test_mandrill_send.py` for examples.



.. _mandrill-response:

Response from Mandrill
----------------------

.. attribute:: mandrill_response

Djrill adds a :attr:`!mandrill_response` attribute to each :class:`~django.core.mail.EmailMessage`
as it sends it. This allows you to retrieve message ids, initial status information and more.

For an EmailMessage that is successfully sent to one or more email addresses, :attr:`!mandrill_response` will
be set to a ``list`` of ``dict``, where each entry has info for one email address. See the Mandrill docs for the
`messages/send API <https://mandrillapp.com/api/docs/messages.html#method=send>`_ for full details.

For example, to get the Mandrill message id for a sent email you might do this::

        msg = EmailMultiAlternatives(subject="subject", body="body",
                                     from_email="sender@example.com",to=["someone@example.com"])
        msg.send()
        response = msg.mandrill_response[0]
        mandrill_id = response['_id']

For this example, msg.mandrill_response might look like this::

        msg.mandrill_response = [
            {
                "email": "someone@example.com",
                "status": "sent",
                "_id": "abc123abc123abc123abc123abc123"
            }
        ]

If an error is returned by Mandrill while sending the message then :attr:`!mandrill_response` will be set to None.


.. _djrill-exceptions:

Exceptions
----------

.. exception:: djrill.NotSupportedByMandrillError

    If the email tries to use features that aren't supported by Mandrill, the send
    call will raise a :exc:`~!djrill.NotSupportedByMandrillError` exception (a subclass
    of :exc:`ValueError`).


.. exception:: djrill.MandrillRecipientsRefused

    If *all* recipients (to, cc, bcc) of a message are invalid or rejected by Mandrill
    (e.g., because they are your Mandrill blacklist), the send call will raise a
    :exc:`~!djrill.MandrillRecipientsRefused` exception.
    You can examine the message's :attr:`mandrill_response` attribute
    to determine the cause of the error.

    If a single message is sent to multiple recipients, and *any* recipient is valid
    (or the message is queued by Mandrill because of rate limiting or :attr:`send_at`), then
    this exception will not be raised. You can still examine the mandrill_response
    property after the send to determine the status of each recipient.

    You can disable this exception by setting :setting:`MANDRILL_IGNORE_RECIPIENT_STATUS`
    to True in your settings.py, which will cause Djrill to treat any non-API-error response
    from Mandrill as a successful send.

    .. versionadded:: 2.0
       Djrill 1.x behaved as if ``MANDRILL_IGNORE_RECIPIENT_STATUS = True``.


.. exception:: djrill.MandrillAPIError

    If the Mandrill API fails or returns an error response, the send call will
    raise a :exc:`~!djrill.MandrillAPIError` exception (a subclass of :exc:`requests.HTTPError`).
    The exception's :attr:`status_code` and :attr:`response` attributes may
    help explain what went wrong. (Tip: you can also check Mandrill's
    `API error log <https://mandrillapp.com/settings/api>`_ to view the full API
    request and error response.)


.. exception:: djrill.NotSerializableForMandrillError

    The send call will raise a :exc:`~!djrill.NotSerializableForMandrillError` exception
    if the message has attached data which cannot be serialized to JSON for the Mandrill API.

    See :ref:`formatting-merge-data` for more information.

    .. versionadded:: 2.0
       Djrill 1.x raised a generic `TypeError` in this case.
       :exc:`~!djrill.NotSerializableForMandrillError` is a subclass of `TypeError`
       for compatibility with existing code.
