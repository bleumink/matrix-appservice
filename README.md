# Modern Matrix Appservices in Rust
This library provides abstractions for building end-to-end encrypted Matrix application services for homeservers also running native OIDC authentication, optionally with state storage.

:construction: **This project is highly experimental and absolutely not ready for production use. Expect major breaking changes, incomplete functionality and sharp edges. Cut yourself at your own risk.**

The library builds upon the [Matrix Rust SDK](https://github.com/matrix-org/matrix-rust-sdk) and ongoing Matrix spec proposals:
- [MSC3202: Encrypted Appservices](https://github.com/matrix-org/matrix-spec-proposals/blob/travis/msc/otk-dl-appservice/proposals/3202-encrypted-appservices.md)
- [MSC4190: Device management for application services](https://github.com/matrix-org/matrix-spec-proposals/blob/quenting/as-device-management/proposals/4190-as-device-management.md)
- [MSC4203: Sending to-device events to appservices](https://github.com/matrix-org/matrix-spec-proposals/blob/tulir/appservice-to-device/proposals/4203-appservice-to-device.md)

## Usage
We're working with the cutting edge here, set the following flags in your Synapse configuration and add the appservice registration file:
```yaml
app_service_config_files:  
  - /data/my_appservice.yaml
experimental_features:
  msc4190_enabled: true
  msc3202_device_masquerading: true
  msc2409_to_device_messages_enabled: true
  msc3202_transaction_extensions: true
```

The appservice registration file needs to following flags set:
```yaml
rate_limited: false
de.sorunome.msc2409.push_ephemeral: true
org.matrix.msc3202: true
io.element.msc4190: true
```

The appservice can then be created as follows:
```rust
#[tokio::main]
use matrix_appservice::{ApplicationService, ApplicationServiceBuilder, EventContext};

async fn main() -> anyhow::Result<()> {
    tracing_subscriber::fmt::init();
    let cli = Cli::parse();
    
    let appservice = ApplicationServiceBuilder::new()        
        .configuration_file(&cli.config)        
        .build()
        .await?;

    appservice.add_event_handler(on_room_member).await?;

    if let Err(error) = appservice.run().await {
        tracing::error!("Application service encountered an fatal error // {}", error);
        return Err(error.into());
    }

    Ok(())
}

async fn on_room_member(
    event: StrippedRoomMemberEvent,
    appservice: ApplicationService,
    context: EventContext,
) -> anyhow::Result<()> {
    let user = appservice.get_bot().await?;
    if event.state_key != user.id() {
        return Ok(());
    }

    // Auto-join on room invite
    match event.membership_change(None) {
        MembershipChange::Invited => user.join_room(&context.room_id).await?,
        _ => (),
    };

    Ok(())
}
```

## :construction: Work in progress
This project is obviously not done; the following is on the to-do list:
- [ ] Documentation
- [ ] Tests
- [ ] Implementing the full Matrix Application Service API
- [ ] Postgres support for state and crypto storage
- [ ] Etc...

Much of the project can also be deprecated by implementing the MSCs mentioned above in the Matrix Rust SDK to provide device masquerading support. I did not know enough about Rust when starting this project to make a sensible contribution there. Might revisit. You would still need to bring your own webserver and handle transactions.