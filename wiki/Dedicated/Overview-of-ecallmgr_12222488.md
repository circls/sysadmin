This is a mailing list post made in July 2012. As such, parts may be unreliable, changed, removed, etc. When in doubt, Read The Source! This page may be prettied up to provide a better overview of ecallmgr's functions, interactions with FreeSWITCH, AMQP, etc.
On to the mail:
        There are a lot of components to ecallmgr. It has grown more
        organically than other pieces of the stack, so things are not as
        tightly contained as we might like (though its getting better with
        each release).

        That said, here's a basic overview to hopefully get you going.

        ecallmgr_fs_nodes is the manager of adding/removing connections to
        mod_erlang_event C-nodes. When we connect to mod_erlang_event, we bind
        to it in a couple different processes:

        1. ecallmgr_fs_node -
 monitors the FreeSWITCH node itself, binds to
        some event types (CHANNEL_CREATE, CHANNEL_DESTROY, some CUSTOM
        variations, etc).

        2. ecallmgr_fs_authn, ecallmgr_fs_route -
 these bind to directory and
        dialplan events, similar to mod_xml_curl. So when FreeSWITCH receives
        a REGISTER or non-ACL INVITE, FreeSWITCH sends a directory request
        proplist to ecallmgr_fs_authn, which converts the request into the
        authn_req AMQP request. The response is received off AMQP and
        converted to directory XML for FreeSWITCh to use.
        Similarly, ecallmgr_fs_route receives dialplan requests. Generally
        speaking, responding whapps just ask ecallmgr to park the call. We do
        a three stage process to determine what whapp 
wins
 the route
        request: ecallmgr_fs_route sends out a route_req; route_resps are sent
        back; the first resp to be accepted by FreeSWITCH gets a route_win
        payload sent back directly to it. In that payload contains the AMQP
        queue of the controller for the call.

        Now, we have a call that has come in. This is where
        ecallmgr_call_events and ecallmgr_call_control come in.
        ecallmgr_call_events binds to FreeSWITCH for the call, meaning all
        call events related to the UUID are sent to ecallmgr_call_events. This
        process (one per call) converts the proplist into call_event AMQP
        payloads. This is how other listeners can know where the call is and
        what the caller is doing (DTMF presses, etc).

        ecallmgr_call_control maintains a queue of commands to run on the
        channel. In the route_win payload from above, the Control-Queue is the
        AMQP queue of the ecallmgr_call_control process. So, if I win the
        route and publish a 
answer
 AMQP JSON payload, I will publish it to
        the targeted exchange using the Control-Queue as my routing key. It is
        possible to share the Control-Queue with others, but its up to the
        application to do the sharing.
        ecallmgr_call_control is also a consumer of call events (published by
        ecallmgr_call_events). It uses the CHANNEL_EXECUTE and
        CHANNEL_EXECUTE_COMPLETE events to determine when to forward its
        command queue.

        On top of ecallmgr_route_req, there's also an ecallmgr_authz_req (run
        concurrently with ecallmgr_route_req). Authorization is optional for a
        call, but when turned on, this module publishes an authz_req and
        expects and authz_resp in return. If the call is authorized, the
        module sends an authz_win to whichever process sent the authz_resp
        first. See the jonny5 whapp for using the authz features (jonny5, from
        the movie Short Circuit).

        freeswitch.erl is just a wrapper module over interacting with
        mod_erlang_event. Because its a C-node, there are several registered
        processes it emulates which we send various types of messages to in
        order to get things done. Its provided along with mod_erlang_event as
        a convenience module.

        ecallmgr_fs_pinger processes get started when we lose connection to a
        known FreeSWITCH node, and will ping it periodically until the
        connection is re-established or you manually rm_fs_node the node. We
        tear all associated processes (ecallmgr_fs_route, authn, etc) down
        when we lose the connection; when pinger successfully pings the
        FreeSWITCH node again, all of the processes are started back up fresh
        (with some syncing done between FreeSWITCH and ecallmgr in case it was
        a temp network outage).

        ecallmgr_fs_resource mostly listens for origination requests (click to
        call, outgoing fax, etc).

        ecallmgr_conference_listener helps manage conference requests (finding
        which FS server a conference was started on, so a caller can be placed
        in it), and ecallmgr_conference_command is the command queue for the
        conference (like ecallmgr_call_command, but with conference-specific
        AMQP APIs).

        ecallmgr_shout and ecallmgr_media_registry help with playing URLs.
        They send media_req out, and receive media_resps with HTTP URIs of the
        .wav/.mp3 files. We then use mod_http_cache and playback to fetch,
        cache, and play the file locally from the FreeSWITCH server.
        ecallmgr_shout handles the recording of calls (voicemail) and having
        FreeSWITCH stream the resulting file to this ecallmgr_shout process.
        This process then can stream the media file up to BigCouch. This
        module will probably be going away, as we can now PUT/POST directly
        from FreeSWITCH using mod_http_cache (http_put). We currently do this
        with storing received faxes; it will be trivial to change the store
        recording command to do the same (just need the time).

        ecallmgr_registrar keeps a local cache of successful registrations
        (though the running registrar whapps are generally the canonical
        authorities), used when sending information to known devices (like
        bridge commands).

        The rest are supervisors or utility modules. You can use appmon to see
        the layout of the supervisor tree.
 
 
The following are the services running in the ecallmgr server:
gen_listener
===================
ecallmgr_call_control
 
ecallmgr_call_events
 
ecallmgr_conference_listener
 
ecallmgr_fs_notify
 
ecallmgr_fs_query
 
ecallmgr_fs_resource
 
 
gen_server
===================
ecallmgr_fs_auth
 
ecallmgr_fs_config
 
ecallmgr_fs_node
 
ecallmgr_fs_nodes
 
ecallmgr_fs_pinger
 
ecallmgr_fs_route
 
ecallmgr_media_registry