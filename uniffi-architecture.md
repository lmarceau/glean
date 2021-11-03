Going forward most of the Glean initialization will happen inside the
Rust core implementation, where it runs in a separate thread.
Callbacks are used to trigger additional actions on the foreign-language
side where appropriate.

The sequence of operations is as follows:

(The following is in some pseudo-code, one statement per line.)
It starts on the foreign-language side:

    Glean.initialize:
        if !ui thread:
          abort

        if !main process:
          log
          return

        if is initialized:
          return

        // Expand user configuration with core config settings
        cfg = buildCfg(from: user_configuration)
        // Gather client info, such as app build, ...
        client_info = buildInfo()
        // Prepare callback object
        callback = Callbacks(
          on_initialize_finished: {
            LifecycleObserver.start()
            Glean.initialized = true
          }
          trigger_upload: {
            PingUploadWorker.enqueue()
          }
          start_mps: {
            MetricsPingScheduler.schedule()
          }
        )

        // Calls the Rust side
        glean_initialize(cfg, client_info, callback)

On the Rust side:

    Glean.initialize:
        Thread.spawn:
            // database init, etc.
            GleanCore.init()

            // Apply early set of operations
            set_debug_view_tags
            set_log_pings
            set_source_tags

            reset_dirty_flag
            register_builtin_pings
            register_custom_pings

            if first run?
              init_core_metrics(client_info)

            glean_on_ready_to_submit_pings()
            // Trigger uploader on early pings
            callback.trigger_upload()

            callback.start_msp()

            if !first run & dirty?
              submit baseline-dirty-startup ping
              callback.trigger_upload()

            if !first run?
              clear_app_lifetime_metrics()
              init_core_metrics(client_info)

            flush_rlb
            flush_queue

            callback.on_initialize_finished()
