# session resumption

I always forgot where are session resumption code lives, so I'm putting that information here.

In `s2n_server_new_session_ticket.c`, the session ticket is written by `s2n_tls13_server_nst_write`. This just writes must of the upper level bytes.

The real interesting stuff happens in `s2n_resume.c` at `s2n_resume_encrypt_session_ticket`.

To get per-sni STEKs, I think we could just change `s2n_resume_generate_unique_ticket_key`. Like we already do the HKDF, we just need to shove the SNI into that.

This means that the client-sni would have to be plumbed into `s2n_server_nst_write` and `s2n_tls13_server_nst_write`.

Hmm, maybe we store the client supplied SNI? That would be very convenient for me.

Okay, there is already a `server_name` field on the connection.
```c
int s2n_conn_find_name_matching_certs(struct s2n_connection *conn)
{
    if (!s2n_server_received_server_name(conn)) {
        return S2N_SUCCESS;
    }
    const char *name = conn->server_name;
```
^ definitely looks like the server name supplied by the client. I guess this is probably just both? E.g. the server connection uses this to store the client's SNI, and the client uses it to store the configured SNI?