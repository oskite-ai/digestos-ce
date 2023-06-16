# Digestos Execution layer
Digestos is a JSON syntax execution language aimed at orchestrating chained AI predictions and making decisions based on the quality of those predictions.

In other words, digestos is kinda an operating system or runtime environment for AIs. Another approach is to think about LLMs as `assembly` code, and digestos would be `high level` coding, but instead of just 1 lang you get to use any lang that intregrates with an AMQP message broker.

Normal APIs usually provide definitive responses. However, in the case of AIs, the response is a prediction with an accuracy score. Furthermore, these predictions may not always be the same, even if the prompt is identical. This implies that we should be able to test or challenge these responses based on the user's expectations. This is done in the Happiness layer. If the output of a chomper does not meet the qualification criteria, then the execution can improve itself before proceeding.

All data is stored in a generic object store, but it is categorized into two types: Persistent and Volatile. The latter implies that you must resend or revalidate volatile data to prevent it from expiring.

Why volatile data? One of the greatest scaling challenges is to "fail early instead of succeed late." This system is designed to gracefully fail and recover any pending tasks.

Adopting a volatile-first approach also allows for two crucial aspects of Digestos:
```
1. The system can mutate by itself
2. The system can extend itself with LLM's generated code
```
Once you publish all the plugins, evaluators, profiles, chompers, and chains, if the process is set to be mutable or extensible, the chain will be altered at runtime, differing from the original version you sent. In other words, all you will be doing is sending a "seed" or "originating" chain, but Digestos may modify it. We will refer to this pack of volatile data as the seed.

It is essential to save audit trails at every step of the process to understand the current status.

You can always download the runtime chain to obtain the current state of volatile data, allowing you to work on it and create a new seed for execution.

## Execution plan
Defines how the work is going to be done
```
[
    {
        "meta": {
            "mutable": false,
            "extensible": false,
            "nestingLimit: false,
            "hThreshold: false
        }
    },
    chomper1,
    chomper2
]
```
### Where
```
- mutable: Chompers are allowed to use the LLM to work on profiles and achieve happier results.
- extensible: special chompers can write code and append nodes in the chain (a chain could start with just one chomper that will extend the chain writing its own chompers to achieve something)
- nestingLimit: how many times can a chomper call itself again
- hThreshold: how happy the response has to be in order to continue executing
```
## Orchestrator
Creates the actual execution chain, adding the input and output nodes, as well as the final output location. This chain is stored using the request guuid64 (called anchor)
```
anchor_guuid64 = [
    {
        "meta": {
            "mutable": false,
            "nestingLimit: false,
            "hThreshold: false
        },
        "tail": guuid64_A,
        "previous": guuid64
    },
    {
        "in": guuid64_A,
        "out": false | guuid64_B,
        "link": link1_guuid64, 
        "c": chomper1
    },
    {
        "in": false | guuid64_B,
        "out": false | guuid64_C,
        "link": link2_guuid64,
        "c": chomper2
    },
    {
        "final": false | guuid64_C
    }
]
```
### Where
```
- anchor_guuid64: where to store the output of the orchestrator
- tail: chain input
- previous: chains are not updated, they are added, and a reference to the previous old version is kept
- in: where the input information for the chomper is saved
- out: where the chomper should write the output data
- link: chomper execution information
- c: what chomper to run (its guuid64)
```
### In and out structures
```
{
    "contex": "...",
    "data": "..."
}
```
## Object store
All data is saved in a no-sql type database.

In order to reduce garbage in the object store, data is split in two categories: persistent and volatile.

Volatile data expires after some time, usually TTL depends on the contract of the user uploading volatile data.

The reason behind this distinction is based on many factors
```
1. To assume that things will fail, gets you writing code that can fix itself
2. After several executions of a complex chain, a lot of context data is going to be created, but becase we don't overwrite data, we tend to hoard a lot
3. When things go wrong, it is very helpful to have all the historical state of things
```

## Chompers
Chomper is what we call the ready-to-execute processes that combine a plugin with a profile, recive an input, produce an output, check the happiness of it and either run again, or pass the output to the next link in the chain.

The first thing a chomper does when they receive a task is to propose a schedule. This is achieved by adding a random number to a list, waiting a few seconds and checking the list again. If other workers are trying to grab the same task, they will honor the random numbers list, the first chomper continues to work, and the rest of the chompers discard the task. If a chomper timesout, the orchestrator will release the task so any available chompers can grab it.
This communication between chompers happens inside the `link` addresses

A special type of chomper is the `extender` chomper, because if allowed, it would add links to a chain.

Given the correct profile and plugin, a chomper could also write a new plugin, and create a chomper to use it, the system extending itself. This is the `extensible` property in action.

Second thing a chomper does is to evaluate the happiness of the result, using the `happiness formula`.

There are 5 categories of chompers
```
1. Input Context: if the user prompt is not enough, or there is no user prompt for the current execution, an input context chomper will ask some service (API, DB, etc) for data
2. Execution Context: parse the input context to prepare whatever context will be needed forward
3. Content Creation: actually generate unformatted content using the given context and profile
4. Content Formatting: shape and stylize the content created
5. Output Delivery: do something with the output of the execution
```
There is some natural order involved, but you could have many stages for each chomper category, and during execution you could jump back, or repeat some chomper category at any place.

Nevertheless, you will start with input chompers and end with output chompers.
## Linking
Here chompers decide which one is going to run a task, as well as saving executing statics and debugging information
```
link_guuid64 = {
    "avail": {
        chomper_worker_guuid: random_int,
        chomper_worker_guuid: random_int
    },
    "stats": [
        {
            "run_id": guuid64,
            "type": execution | evaluation,
            "start": time,
            "end": time,
            "sucess": bool,
            "message": debug information,
            "spent": {...}
        }
    ]
}
```
Where
```
- avail: decide the order of the workers
- stats: each execution adds an item to the array with profiling and debugging information
```
## Plugins
A chomper is the actual combination of a plugin and a profile.
Plugins expose the next interface so you can connect to any service
```
build_request(context, prompt, profile)
execute()
normalize_expenditure(spent)
```
The actual lang of the plugin does not really matter, because it will be matched with a cook/worker for that lang (see: polyglot)
## Profiles
In the profiles user can define how the chomper should use the plugin. Remember the chomper it is nothing more than a configuration saying `i will want to execute this plugin with this profile applied`

So the profile needs to have some information. This is how a profile would look like
```
{
    "meta": {
        "auth_secret": guuid64
    },
    "profile": {
        "preprompt": string | uuid,
        "postprompt": string | uuid,
        "flavour": string | null,
        "output": {
            "format": json | text | binary,
            "shape": {...}
        }
    }
}
```
Where
```
- preprompt and postprompt = actual value or stored prompt
- flavour = goes right before post prompt, for stylizing
- shape = HxW, JSON dict, or null if plain text
```
## Expenditure
Each API has their own way of calculating costs, so each plugin should be able to normalize expenditure to the following standard format
```
{
    "req_count": integer,
    "req_cost": float,
    "rem_bal": float,
    "auth_guuid": guuid64
}
```
Where
```
- req_count = how many requests to the API the chomping did
- req_cost = averaged cost in USD per request for this API at the time of doing these requests
- rem_bal = remaining balance in this API for the executed profile's auth_guuid
- auth_guuid = public id (you can only use it to group requests, not to retrieve credentials) of the credentials needed to request stuff to some API
```
The expenditure is saved in `requests` entity.
# Digestos Auth Layer
Digestos uses customer-based json-like contracts. This approach allows for a stack-wide description of the behaviour expected by the user. Think of this method as an extreme feature-toggles approach.

This is how a final contract might look like
```
{
    "priv": {
        "plugin_auth_addr": guuid64,
        "priority": [low,high],
        "store_url": [subdomain,path]
    },
    "pub": {
        "dig_url": [subdomain,path],
        "lp_url": [subdomain,path],
        "meta": metadata_guuid64
    }
}
```
The `priv` key holds all the information we need to make the backend work as expected, while the `pub` key holds the information that will be sent in the auth JWT token.
```
- plugin_auth_addr: where are the credentials for each plugin for this user
- priority: run something ASAP or just joing the low priority queue
- store_url: endpoint to read and write stuff
- dig_url: where to post prompts
- lp_url: where to request responses (longpolling)
- meta: store address of the additional public contract information
```
## We need 7 entities to handle users and their contract claims

```
codes(code,expiration_date,start_date,campaign_id,claimed)
codes_traits(code_id,trait_id)
campaigns(title,slug,enabled,trait_id,claim_counter)
users(api_key,api_secret)
users_claimed_codes(user_id,code_id,campaign_id,claimed_date,contract,contract_version,terms_signature)
contract_traits(title,internal_tag,contract,version,parent_trait_id,enabled)
contract_terms(title,description,internal_tag,enabled)
```
## Campaigns
Define the funnels. You can use different campaings for versioning, A/B testing, etc.
```
- title: descriptive title of the funnel
- slug: a.k.a. contract-code (users claim traits using slugs or straight up codes)
- enabled: should users be able to claim codes in this campaign
- trait_id: end node in the contract chain, from this ID the contract is formed by going upwards the parent trait id
- claim_counter: how many users claimed codes from this campaign
```
## Codes
Pre-generated guuid64 codes that keep the track of the user's contract trait claim. You could also just generate the code during user registration.
```
- code: guuid64
- expiration_date: until when this code can be claimed
- start_date: since when this code can be claimed
- campaign_id: to wich campaign this code belongs to
- claimed: you can only claim a code once
```
## Codes traits
While campaigns should not change their `end of node` trait id, it might happen after bug corrections or specific deployments. It is important to store the current trait_id when a code is added to the codes tables, so if the campaign table changes, the previusly generated codes will still base their contracts from the correct trait (iow, don't break old codes)

## Contract traits
This table holds the json pieces that will form the final user json contract
```
- title: descriptive title
- internal_tag: this string is used to group traits and terms
- contract: piece of json object that holds the trait rules/settings
- version: incremental int for each contract change, but traits should not be overwritten. You add a new row with the new version so we can sort it later
- parent_trait_id: the path upwards the base contract
- enabled: bypass this trait
```
This is how a contract trait might look like
```
{
    "priv": {
        "priority": "high"
    }
}
```
So if the base contract has a value of `low` for the `priv.priority` key, when a user claims a code that links this specific trait, the final contract will have the value `high` for that key, instead of the default.

This allows you to mix up many different versions of the same app, makes A/B testing super easy, and makes it easier to onboard users into restricted areas.

Other traits could be:
```
- Addresses of the services you will want to connect to
- Actual URLs to post or get info from backend
- Customer branding information
- Quotas and throtling
```

## Contract terms
`terms` are just a block of text describing the contractual policies.

Depending on the `contract_version` label and the `internal_tag` of a user claimed code, these terms will be grouped and have to be accepted by the users. We then save a signature of the accepted terms in `terms_signature`, if some term is deprecated and/or a new term is added, the signature will change, user will be challenged again to accept the new composed terms

## Users claimed codes
Actual contract for each user
```
- user_id: who owns this contract
- code_id: which code was used to claim this contract
- campaign_id: just repeating the campaign id
- claimed_date: audit info
- contract: aggregated JSON object that is formed starting from a base contract, and then applying all the chained traits for the used code
- contract_version: tokenized string of the version of each trait that was applied to the base contract, in the correct order. In order to reduce the size of the token string, each guuid64 for each trait will be condensed using guuid64/sha1 (keep the first part of the guuid, and combine with first 6 chars of sha1 digestos for entire guuid), so in the case of a collision on the first part of the guuid, we can discriminate using the 6 char signature
- terms_signature: md5 of the concatenations of the contract term ids accepted by the user
```

# Admin api Layer
Configuration is based on these entities
```
Persistent:
- users & contracts
- requests
- prompts (name,description,prompt)

Volatile:
- plugins (name,description,code_bundle,version,category) & plugin_tags
- evaluators (name,description,code_bundle,version) & evaluator_tags
- profiles
- chompers
- chains
```
### Difference between persistent and volatile
Persistent data is always available for rw. Volatile data expires and needs re-validation to stay alive. When sending a work batch you need to send the volatile data. You can re-send the batch-req-id with recurrent batches to re-use volatile data (re-validation is automatic in this case)
# Happiness Layer
Maybe the most important layer of them all. 

Sentiment analysis of the responses, with user feedback, in order to optimize and enhance the prompts. Could also be used to make the responses keep a style or trademarks

Implementation is very straight forward. The profile needs to define evaluators and the happiness formula

## Evaluators
Given the chomper's input, output and metadata information, the evaluators return 0 or 1 (true or false). IOW is the output of the chomp something we evaluate as desirable

Just like plugins, you need to register the evaluator's code, creating the relation with the correct runtime so it can be called.

Evaluators just have 1 method
```
evaluate(input,output): bool
```

## Happiness formula
happiness = evaluator_1_result * weight_1 + evaluator_2_result * weight_2 + ... + evaluator_N_result * weight_N

Where
```
- evaluator_result_N = 0 or 1
- weight_N = a number between 0 and 1, how happy the evaluator being true would make us
```

## Happiness threshold
There is a global happiness and a chomper happiness. If you don't define the chomper specific happiness, global happiness applies.

The threshold is a number between 0 and 100, being 100 absolutely happy.

When calculating the result of the happiness formula, there is a second calculation assuming all evaluators returned true, this sum equals the maximum happiness for that chomper's profile.

Using these values we can calculate if the result happiness is over the happiness threshold.

Possible outcomes when output is not happy enough are
```
- (mutable=no,nestingLimit=0) = chain execution fails
- (mutable=no,nestingLimit>0) = try again until limit or chain execution fails
- (mutable=yes,nestingLimit>0) = use LLM to improve prompt and try again
```

## Happiness integrations
Integrations are a type of input context plugin, but instead of fetching context from somewhere, they actually reach a human for feedback. This way, evaluators can maybe reach out to a stakeholder, present in some sort of way the evaluation the human needs to do, and catch the human response, suggestions, comments, etc, in order to give the evaluator a response we can plug into the happiness formula, and use the human response for something.

For example, an output delivery plugin might first ask for a human code review on a PR and wait for either a confirmation, or instructions about errors or other things that need to be fixed before deployment.
# How it all runs
Let's try to make some analogies
## Orchestrator service: the chef
It prepares the work to be done. Sends out messages into the work queues. Constantly checks for execution progress, frees up locked states.
## RabbitMQ: the waiter
Move things around. Split transport layer by virtual host, or in our case, proyects. Each proyect is isolated from the other, grouped by an objective in common.
### Exchanges
There is one exchange for each chomper, plugin and evaluator.

The routing key is of the format
```
action.priority
```
For the time being, the only action is `run`.

Priority can be `high` or `low`. This is just a grouping for how many workers you want to deploy for each priority group.

### Messages
```
{
    "meta": {
        "auth": [header,pld,signature],
        "version": integer,
        "from": [entity_name,guuid]
    },
    "payload": {
        {...}
    }
}
```
Where
```
- auth: JWT token
- version: msg version identifier
- from: which entity/server sent this message along with their execution id
```
## Queue workers: the cooks
Grab tasks and do stuff

## Kitchens
Allocated availability of cooks ready to do stuff in some cloud

This means workers started and grabbed evaluators and plugins, saved them locally, so they can actually run stuff. 

Both evaluators and plugins should have N tags associated with them, so kitchens need to know the tags they need to look for

## Polyglot
Because RabbitMQ has plenty of language integrations, we can have kitchens with cooks that work on many languages. This way, we are not constrained to any single coding lang, we can have plugins and evaluators basically for any type of architecture and language.

At first sight it might seem overkill, but Digestos is designed to be scale-ready from day zero. If you never scaled up a company, let me tell you, it is a lot easier to find 10 awesome devs that use any of 10 given langs, than finding 10 awesome devs within a single lang or framework.

To succeed in scaling up a software company, you have to be able to mutate your foundations without dying in the process.

## Always available, self-healing and multi-versioning
RabbitMQ excels in availability and dead-letter handling, so the service has 100% uptime.

Because RMQ workers are agnostic about their peers, it is very easy to keep multiple versions of the same action happening at the same time, compare them, benchmark it, etc.

## Virtual Hosts
RabbitMQ isolates environments by defining a virtual host path. We can use this to group up the different entities that fall under the same objective. The virtual path can be used as a flag in the auth contract. 