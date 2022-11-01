# Feature Name: Topics

:warning: This proposal depends on [one-shot systems](https://github.com/bevyengine/bevy/pull/4090), which is not yet a merged feature, and possibly [trait queries](https://github.com/bevyengine/rfcs/pull/39) which is not yet a merged RFC.

## Summary

This RFC introduces a feature to help with developing reactive applications.
Services are an asynchronous one-to-one request -> response mechanism.
A client submits a request to a service provider and is immediately given a promise.
The service provider processes the request asynchronously and eventually delivers on the promise.

## Motivation

Services are a common mechanism for performing Remote Procedure Calls or querying data over distributed systems, supported by middlewares like HTTP-REST, ROS, ZeroMQ, gRPC, and many others.
Service providers do not actively publish messages.
Instead, a client submits a request, and a service provider reacts to that request and delivers a response at some later point.
Services are asynchronous, which allows the service provider to spend time processing the request as needed, possibly making calls to remote systems, without blocking the rest of the application's workflow.

This RFC proposes implementing services inside of Bevy as ergonomic Systems, Resources, and Components.

Services can be used inside of Bevy to help Bevy users implement their applications using a "distributed" architecture to decouple and organize event flows from various agents/entities.
They are especially helpful for managing request-response data exchanges that may require time to process, other events to occur, or remote calls before a response can be given.

At the same time, defining this generic ergonomic pipeline for services within Bevy should help facilitate the integration of various middlewares into Bevy.
For example, an `HTTP-REST` plugin could simply be implemented as a generic service provider within Bevy.
Bevy systems can make service requests without being concerned about whether the service provider is local or in another process or on a remote machine.
The Bevy systems can also be agnostic to what kind of middleware is being used to fulfill the request, allowing a Bevy application developer to test out different middleware options and mix-and-match without any changes to how their Bevy systems are implemented.

## User-facing explanation

These are the key structs for services:
* `Service<Request, Response>`: a component or resource that represents a type of service that can be provided, defined by a `Request` input data structure and a `Response` output data structure.
The description can be used by clients to decide which service provider they want to use for a given service type.
* `Promise<Response>`: a component for awaiting the result of a service request that has been issued.
A `Promise` can be polled to check if the response is available yet.
Alternatively it can be loaded with a callback system which will be queued as soon as the response is available.
* `Domain`: a component that allows service providers to advertise their services so that potential clients can find them.
* `GlobalDomain<T>`: a resource that can be used as a service domain.

### Making Requests
To request a service, use the `Service::request` method:

```rust
pub struct RequestPath {
    pub from_place: Place,
    pub to_place: Place,
}

struct SolvedPath(pub Vec<Place>);

type PathSearch = Service<RequestPath, SolvedPath>;

fn request_a_path(
    commands: Commands,
    path_service: Query<&PathSearch>,
) {
    let path_promise = path_service.single().request(
        RequestPath {
            from_place: Place::NEW_YORK,
            to_place: Place::BOSTON,
        }
    );
    commands.spawn().insert(path_promise);
}
```

Note that an entity is spawned to hold onto the Promise.
When a Promise is dropped, the Service will no longer be expected to fulfill it.
You could also store the promise as a field in a component or resource to prevent it from dropping.

If you don't care about receiving the response, you can call `Promise::release` which will tell the service to fulfill the Promise even though it is being dropped.
`release` is especially useful after you have attached a callback to your promise.
For example:

```rust
pub struct Arrival {
    pub place: Place,
    pub time: Time,
}

type Drive = Service<SolvedPath, Arrival>;

#[derive(SystemParam)]
struct DriveParam<'w, 's> {
    commands: Commands<'w, 's>,
    driver: Query<'w, 's, &Drive>,
}

fn request_a_path(
    path_service: Query<&PathSearch>,
) {
    path_service.single()
        .request(
            RequestPath {
                from_place: Place::NEW_YORK,
                to_place: Place::BOSTON,
            }
        )
        .then(
            |response: SolvedPath, param: DriveParam| {
                let drive_promise = param.driver.single().request(response);
                param.commands.spawn().insert(drive_promise);
            }
        )
        .release();
}
```

In the above example, a `PathSearch` service is requested to find a path from New York to Boston.
Once the path is found, a `Drive` service is requested to follow the path.
The promise of the `Drive` service is saved as a component of a newly spawned entity so that it can be monitored and perhaps cancelled by dropping the promise later.
The outer promise is released because we are not interested in monitoring or cancelling it.

Using `.then` on a Promise will consume the original Promise and give back a new Promise whose "promised value" matches the return type of the inner function.
For example:

```rust
fn find_path_cost(
    commands: Commands,
    path_service: Query<&PathSearch>,
) {
    let cost_promise: Promise<Cost> = path_service.single()
        .request(
            RequestPath {
                from_place: Place::NEW_YORK,
                to_place: Place::BOSTON,
            }
        )
        .then(
            |response: SolvedPath, cost_calculator: Res<PathCostCalculator>| {
                cost_calculator.calculate(response)
            }
        );

    commands.spawn().insert(cost_promise);
}
```

The promised value that we are left with after calling `.then` will be the return value of the callback after its system has been run with the response of the original service.

If the return type of the callback is itself a `Promise<T>` you can instead use `.and_then` to flatten the outer promise to a `Promise<T>` instead of being a `Promise<Promise<T>>`:

```rust
fn drive_to_place(
    commands: Commands,
    path_service: Query<&PathSearch>,
) {
    let arrival_promise: Promise<Arrival> = path_service.single()
        .request(
            RequestPath {
                from_place: Place::NEW_YORK,
                to_place: Place::BOSTON,
            }
        )
        .and_then(
            |response: SolvePath, driver: Query<&Drive>| {
                // Returns a Promise<Arrival>
                driver.single().request(response)
            }
        );

    commands.spawn().insert(arrival_promise);
}
```

To track whether a promise has been fulfilled, you can poll the promise:

```rust
fn watch_for_arrivals(
    commands: Commands,
    driving: Query<Entity, &mut Promise<Arrival>>,
) {
    for (e, arrival) in &mut driving {
        if let PromiseStatus::Resolved(arrived) = arrival.poll() {
            commands.entity(e).insert(arrived);
            commands.entity(e).remove::<Promise<Arrival>>();
        }
    }
}
```

The `PromiseStatus` returned by `Promise<T>::poll` is an enum that lets you track the status of your promise:

```rust
pub enum PromiseStatus<T> {
    /// The Promise is being processed by the service provider
    Pending,
    /// Receive the result. This can only be received once in order to support
    /// cases where T does not have the Clone trait.
    Resolved(T),
    /// The result has been consumed by an earlier call to `Promise::poll` and
    /// can no longer be obtained.
    Consumed,
    /// The service provider has dropped, so the Promise can never be fulfilled.
    Dropped,
    /// The service provider experienced an operational error (e.g. lost
    /// connection to remote service) so the Promise can never be fulfilled.
    Failure(anyhow::Error),
}
```

### Providing Services
To implement a service provider, create a system that queries for the service to check for unassigned requests.
For convenience, you can assign a `bevy_tasks::Task`, at which point it will no longer show up in the unassigned list.
It will remain in the pending list until the task is completed.
The `SearchType` description helps inform how the service should behave.

```rust
pub enum SearchType {
    /// Search for the optimal path
    OptimalPath,
    /// Find any valid path as quickly as possible
    QuickSearch,
}

fn handle_path_search_requests(
    path_services: Query<&mut PathSearch>,
    planner: Res<Arc<PathPlanner>>,
    pool: Res<AsyncComputeTaskPool>,
) {
    for service in &mut path_services {
        let search_type = service.get_description::<SearchType>().unwrap_or(SearchType::Optimal);
        for unassigned in &mut service.unassigned_requests() {
            let planner = planner.clone();
            let search_type = search_type.clone();
            if let Some(request) = unassigned.take_request() {
                unassigned.assign(
                    pool.spawn(async move {
                        planner.search(request, search_type);
                    })
                );
            }
        }
    }
}
```

Once the task is completed, the request will no longer be in the pending list, and the next time its Promise is polled, it will return `Resolved(T)` with the result that was given by the task.

Alternatively, if a task cannot encapsulate the work that needs to be done for the service, a system can directly resolve pending requests:

```rust
fn detect_arrival(
    drivers: Query<&mut Drive>,
    current_place: Res<Place>,
    mut current_path: ResMut<Vec<Place>>,
    clock: Res<Clock>,
) {
    for service in &mut drivers {
        for unassigned in &mut service.unassigned_requests() {
            if let Some(request) = pending.peek_request() {
                current_path.extend(request.iter());
            }
            // This request will no longer show up in the unassigned list, but
            // will still be visible in the pending list.
            request.assigned();
        }

        for pending in &mut service.pending_requests() {
            if let Some(request) = pending.peek_request() {
                if request.0.last() == Some(*current_place) {
                    pending.resolve(Arrival {
                        place: current_place.clone(),
                        time: clock.now(),
                    });
                }
            }
        }
    }
}
```

### Discovery
The `Domain` can be used to advertise the service of an entity:

```rust
fn make_path_search_service(
    commands: Commands,
    domain: ResMut<GlobalDomain<MyDomain>>,
) {
    let optimal_search = commands.spawn().insert(
        PathSearch::new().with_description(SearchType::OptimalSearch)
    ).id();

    let quick_search = commands.spawn().insert(
        PathSearch::new().with_description(SearchType::QuickSearch)
    ).id();

    domain.advertise::<PathSearch>(optimal_search, "driving_path".to_string());
    domain.advertise::<PathSearch>(quick_search, "driving_path".to_string());
}
```

Then a client can discover these services and select one of them based on the description:

```rust
fn find_optimal_path_search_service(
    domain: Res<GlobalDomain<MyDomain>>,
) {
    let service = domain.discover::<PathSearch>("driving_path".to_string())
        .filter(|service| {
            service.get_description::<SearchType>()
                .filter(|d| *d == SearchType::Optimal)
                .is_some()
        })
        .next();

    if let Some(service) = service {
        service.request(
            RequestPath {
            from_place: Place::NEW_YORK,
            to_place: Place::BOSTON,
        })
        .and_then(
            |response: SolvePath, driver: Query<&Drive>| {
                // Returns a Promise<Arrival>
                driver.single().request(response)
            }
        )
        .release();
    }
}
```

## Implementation strategy

Many details of this implementation proposal will need significant iteration with help from someone more familiar than I am with the internals of Bevy.
I am currently not clear on how exactly dynamic system construction can/does work, nor the right way to leverage one-shot systems to make this work.
Chaining promises may be especially challenging.

### Promises
* `Promise<Response>` instances are created by `Service<Request, Response, ...>` instances
  * Channels are created between the promise and the service using `crossbeam`
  * When the result is ready, the service will send it over a channel
  * When a callback gets chained to a promise, it uses a different channel to push the callback to the service
* The promise will maintain a `PromiseStatus` to track its current state
  * When the user calls `Promise::poll(&mut self)`, the `PromiseStatus` is consumed.
  * If the `PromiseStatus` was `Resolved(T)` then the next `PromiseStatus` will be `Consumed`
  * For all other `PromiseStatus` values, the next value will be a clone of the last.
  * The user can `peek_status(&self)` instead to get a reference to the status and not consume it.

### Services
* Services keep track of requests that have been issued to them
* Requests that are being tracked by a service can have an "unassigned" status which indicates that the service has not started working on it yet
  * All new requests are automatically set to unassigned
  * A request can be assigned to a `bevy_task::Task` to automate its fulfilment
* Requests that have not been fulfilled yet will have a "pending" status (all unassigned requests are also pending)

### Domain management

The domain keeps track of which topics the service providers want to be discovered on and what type of service they provide for that topic.
This will likely be a HashMap to a HashSet of entities.

The domain will also need to store "service stubs" for all the services so that users can call `request` on the discovered services without needing to Query for the service components.
Maybe this can be done by sharing an `Arc` between the real service component and the stub that's held by the domain.

## Drawbacks

Many aspects of this proposal might be very difficult to implement. There are currently many gaping holes in the implementation proposal.

It might be necessary to use [trait queries](https://crates.io/crates/bevy-trait-query) to implement some aspects of this proposal.

## Rationale and alternatives

It is often difficult and awkward for users to figure out how to structure their code when they need to handle futures that require polling.
Services attempt to eliminate the need for users to think in terms of futures by giving a simple way for them to describe the entire asynchronous data flow that they want in a single location.
Being able to chain reactions is especially valuable for dealing with complex multi-stage data flows.

This design pattern also cuts down on the boilerplate that users need to write for dealing with asynchronous programming.
They will no longer need to write any system whose entire job is to poll for tasks that are running inside of task pools.
Instead usages of task pools can be written more ergonomically as services.

### Service Descriptions
In the proposal, generic descriptions are stored inside of the `Service` and queried by clients.
This is done to keep the description information encapsulated inside of the `Service` component.

Should these descriptions actually be saved as separate components on the entity instead of buried inside of the `Service` component?
That approach would align better with an ECS design philosophy but might be less convenient for users.

## Prior art

The ergonomics of promises proposed here is very similar to JavaScript promises, which I think are very effective at expressing asynchronous data flow (even if I have less positive opinions about the JavaScript language more generally..).

We could draw inspiration from other implementations of Promises that have been done in Rust [[1]](https://docs.rs/promises/0.2.0/promises/struct.Promise.html)[[2]](https://docs.rs/promises/0.2.0/promises/struct.Promise.html). It's not clear to me if any of those crates can be used directly or if we need to customize the implementation too much because of the ECS backbone.

## Unresolved questions

### Network / Connection Errors
What kind of impossible-to-deliver errors should be supported, and how should they be supported?
For example, if a service needs to make remote call but no network is available, is `PromiseStatus::Failure(anyhow::Error)` adequate?
The `Response` type of the service could be a Result with a user-defined error type, but I think it would be nicer if the `Response` type of the service should only contain domain-specific concerns related to the `Request` and should not care about service pipeline issues.
Otherwise the `Response` type becomes sensitive to the choice of service middleware (or lack thereof).

### Service Progress
For services that take a long time to finish, a user may want to be able to track the progress that is being made.
I can think of a few ways to do this:

* Provide a new type like `Action<Request, Response, Progress>` which mimics the [ROS Action semantics](https://docs.ros.org/en/foxy/Tutorials/Beginner-CLI-Tools/Understanding-ROS2-Actions/Understanding-ROS2-Actions.html).
  * Advantages: Clear semantic distinction from `Service`, which will only provide a Response
  * Disadvantages: A lot of functional duplication with `Service`
* Add a `Progress` generic to `Service` with a default of `()`, i.e.: `Service<Request, Response, Progress=()>`.
  * Advantages: No functional duplication and easy to ignore the Progress capability if it's not needed
  * Disadvantages: Services providers that provide different `Progress` types will be categorized as entirely different service types even if they have the same (`Request`, `Response`) pair
* Use type erasure to hide progress types inside of the `Service<Request, Response>` similar to the service description. Clients can query the `Service` to see what types of Progress updates it supports.
  * Advantages
    * Any service providers with the same (`Request`, `Response`) pair will be discoverable together
    * A service provider can support multiple types of `Progress` updates
    * A client can receive different multiple types of `Progress` updates from the same service
  * Disadvantages: harder to implement?

### Chaining a promise after it is fulfilled

If a promise has been fulfilled, it should still be possible to chain a new callback to it, as long as the response has not been consumed.
Somehow the existence of the new stage needs to get packaged up with the response data and processed by the service systems.
