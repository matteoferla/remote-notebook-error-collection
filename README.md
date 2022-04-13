# remote-notebook-error-collection

> :construction: WIP: This is a MVP prototype.

This aim to collect errors generated by other users using a notebook that was shared.

I have a few notebooks that I have shared on Twitter and 
I occasionally get an email telling if the repo they use is broken
or there is a case that causes an error.
Similar in concept to Sentry.io, I would like to know when error happen.

I do not want any private or confidential data from the user or user given fields
—someone's target protein might be confidential.

I only want to send the error type, the error message, some traceback details,
and the notebook name and the cell.

The code therefore should not contain error codes raise someone's password or credit card number.
In the scientific context I don't think this should be a problem.

In a colab notebook this is rather straightforward.
Data I would avoid collecting are browser's IP, inputted values and majorly content of a Google Drive if mounted.

Three classes presented here are successive steps in its construction.

## Store

An alternative option is storing the error details `error_details`.
```
from notebook_error_reporter import ErrorStore
es = ErrorStore()
es.load_ipython_extension()
es.error_details
```


## Slack
The easiest way is getting slacked on error to a channel.
A Slack webhook is easy to set up (just remember the subdomain to do so is api not app).

```python
import os
os.environ['SLACK_WEBHOOK'] = "https://hooks.slack.com/services/XXXXXXXX"

from notebook_error_reporter import ErrorSlack
es = ErrorSlack(os.environ['SLACK_WEBHOOK'])
es.load_ipython_extension()
```

A regular cell does nothing. But one that is not successful will send a Slack message.

    {"error_name": "ValueError", 
     "error_message": "foo", 
     "traceback": [{"filename": "foo.py",
                    "fun_name": "run_code", 
                    "lineno": 666}, 
                    ...
                   ], 
     "first_line": "# cell that does foo",
     "execution_count": 111}

The 'filename' is stripped of the dist-packages path, 
because the `dist-packages` path in colab may have a username that _could_ have personal identifiable data.

If a Slack webhook is shared on GitHub, there are users that search GitHub for exposed webhooks 
and spam with adverts for their cybersecurity courses.
Also a single prankster user could make it really annoying.
Therefore, a server needs to be set up ideally to collect this...

## Server

A FastAPI app to get the errors is also present.
This needs to be set up on a hosting server exposed to the internet.

This has the largest risk of vandalism.

Running `run_app.py`, which contains this code:
```python
import uvicorn
from fastapi import FastAPI
from notebook_error_reporter.serverside import create_db, create_app

create_db()
app:FastAPI = create_app(debug=False, max_transparency=True, colab_only=False)

if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0", port=8000, log_level="info")
```
and activating on the notebook thusly:
```python
from notebook_error_reporter import ErrorServer

es = ErrorServer(url='http://127.0.0.1:8000', notebook='mine')
es.load_ipython_extension()
```

On error a dictionary typehintinted as `EventMessageType` is sent:

```python
from notebook_error_reporter import EventMessageType

EventMessageType.__annotations__
```

    {'execution_count': int,
     'first_line': str,
     'error_name': str,
     'error_message': str,
     'traceback': typing.List[notebook_error_reporter.error_event._traceback.TracebackDetailsType]}

and `TracebackDetailsType.__annotations__` is:

    {'filename': str, 'fun_name': str, 'lineno': int}

The server does keep track of IP addresses to prevent vandalism,
but it's the IP address of the colab notebook. No JavaScript call is present to get the browser IP.
Therefore the IP will be in the range: 142.250.0.0 - 142.251.255.255.

To see the errors sent:

```python
es.retrieve_errors()
```

I am unsure if to allow everyone to see the sessions and errors, hence the `max_transparency` argument.
For an internal server, this makes sense, but for a public one, revealing the session ids may 
result in vandals adding errors to sessions randomly.


