# AdEx validator stack

## In progress

#### This document is currently a work in progress.

#### However, there is an actual reference implementation of the validator stack here: https://github.com/adexnetwork/adex-validator

#### As of the v0.4 milestone (2019-04) of the reference implementation, most of this document is outdated, except Bidding process, which is not part of the validator stack anyway

-----------------------

## Architecture

The validator stack is split in two discrete parts:

* The Sentry handles HTTP(S) requests from the outside world, and communicates with the underlying database (MongoDB in our reference implementation)
* The Worker communicates with the Sentry and periodically generates new signed OUTPACE states

A single validator can be a leader or a follower, in the context of an OUTPACE channel. The architecture is the same in both cases, but the Worker's role is different: a leader is solely responsible for producing new states, while the follower is responsible for signing the states provided by the leader, but only if they are valid.


## Redundancy

The Sentry is a stateless microservice, which means it can scale horizontally. This is usually done for performance reasons, but it also provides added redundancy.

@TODO db abstraction

## Flow

@TODO explain EWT/JWT authentication

@TODO explain what the flow is FOR; all processes that require the validator stack

* negotiate validators
* create a channel (ethereum, polkadot, whatever)
* upload campaignSpec to the market, and potentially to validators (IPFS?)
* each validator would go through these states for a channel: `UNKNOWN`, `CONFIRMED` (on-chain finality), `LIVE` (we pulled campaignSpec and received a `init` msg from other validators) (other states: `EXHAUSTED`, `EXPIRED`, `VIOLATED_CONSTRAINTS`)
* AdView asks all publisher-side platforms for ACTIVE channels, sorts by targeting score, takes top N, then sorts by price and takes top M - then randomly chooses an ad from the remaining, and signs a message using the user's keypair on which campaign was chosen and at what price


@TODO: negotiating the validators MAY be based on deposit/stake

## Components

### Sentry

The Sentry is responsible for all the RESTful APIs and communication.

Furthermore, the Sentry is the point that receives events from users, and it is it's job to impose limits described in the `campaignSpec` (e.g. max 10 events per user) and other sanity checks (e.g. limits per IP). Basically, any event that the sentry puts in the database will be counted by the validator worker.

Multiple Sentry nodes can be spawned at the same time, and they can be across different servers/data centers. This will ensure redundancy and near-100% uptime. Furthermore, even if the OUTPACE worker crashes, as long as the events are recorded, channels will always be able to recover when the worker starts up again.

@TODO nginx, ip restrictions

#### API

@TODO http://restcookbook.com/HTTP%20Methods/put-vs-post/

##### Do not require authentication, can be cached:

GET /channel/:id/status - get channel status, and the validator sig(s); should each node maintain all sigs? also, remaining funds in the channel and remaining funds that are not claimed on chain (useful past validUntil); AND the health, perceived by each validator

GET /channel/:id/tree - get the full balances tree; you can use that to generate proofs

GET /channel/list

##### Requires authentication, can be cached:

GET /channel/:id/events/:user

##### Requires authentication:

POST /channel/events

POST /channel/validator-messages


#### Validator messages

Each validator message has to be signed by the validator themselves

OUTPACE generic:

* `NewState`: proposes a new state
* `ApproveState`: approves a `NewState`
* `Heartbeat`: validators send that periodically; once there is a Heartbeat for every validator, the channel is considered `LIVE`
* `RejectState`: sent back to a validator that proposed an invalid `NewState`

AdEx specific:

* SetImpressionPrice: set the current price the advertiser is willing to pay per impression; also allows to set a per-publisher price

Each message must be individually signed by the validator who's emitting it.


### OUTPACE validator worker

@TODO this is where the signing key is handled; describe how this can work: randomly generated keypair, HSM ?

@TODO describe blockchains-specific adapters

@TODO validator stack: might need a restriction on the max publishers, or on min spend per publisher; since otherwise it might not be worth it for a publisher to withdraw

@TODO describe the problem that a few publishers might chose a channel when there's a small amount of funds left, and this will create a race where only the first impression would get paid; we can solve that by paying in advance, but this is impractical since it requires 2 communication hops (1 commit, 1 payment)

### Reports worker


## Bidding process

@TODO move this out of here

Each campaign has a total budget, and a maximum amount that the advertiser is willing to pay per impression.

Other than that, the advertiser may adjust the amount that they want to pay dynamically, during the course of the campaign. They may also adjust the amount for each individual publisher. This is done by sending a validator message to the publisher-side platform.

On every impression, the AdView (running on the publisher's website/app) will pull all active and healthy campaigns from a configurable list of publisher-side platforms.

Then, it will sort all ad units by targeting score and only leave the best N matches. Then, it will sort the remaining group by the price the advertisers are willing to pay, take only top M (both N and M is configurable by the publisher, see [`topByPrice`](https://github.com/adexnetwork/adex-adview-manager#options)).

By adjusting N/M, the publisher can decide the balance between UX (more appropriate ads shown to the users) and revenue.


## DB structure

* Channels
* Messages
* EventAggregates
