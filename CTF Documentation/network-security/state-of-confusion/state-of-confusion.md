# State of Confusion
Enumerate the gateway and inspect its banner:

HELP
SHOW BANNER

Negotiate the required protocol:

HELLO MUX/2

Discover and use the laboratory guest profile:

SHOW USERS
LOGIN guest guest

Save the returned ticket. Enumerate the channels:

LIST CHANNELS

Attempt to select the operator-only channel:

SELECT CONTROL_CHANNEL

The gateway denies access and clears authentication, but the selected channel remains in session state. Confirm that state:

STATUS

Resume the original ticket:

RESUME ORIGINAL_TICKET

Read the still-selected restricted channel:

READ

The response contains the flag.

flag{inm_resume_does_not_mean_reauthorize}
======================================================================