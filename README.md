# IN DEVELOPMENT.  PRE-ALPHA. THE FOLLOWING README is a vision for the project/wishlist.  

### I feel most of these items are low-hanging fruit, and I feel very strongly this project fills a growing gap. But as much as this might be fast and necessary, it is still a FOSS passion project and I don't have much time to chase my passions ATM.  Check back in a month and maybe you'll see progress, or maybe even a structure to contribute, but even that basic groundwork is going to be slow for now.
![PyralGlo](pyral_glo_full.png)


**Tl;Dr:** Package-level credential validation at instantiation with management baked in. Credential acquisition mechanisms are decrypted, verified, and tested before any business logic runs. Encrypted storage reduces environment variable clutter while adding access points to enable key-rotation rather than adding friction to it. These ansulary additions allow all the benefits of package-level credential management without the downsides.


## Preflight Validation: PyralGlo ensures your package never starts with broken, expired, or invalid credentials.
PyralGlo is a runtime credential validation and management layer.
It encrypts and validates your credentials, whether stored locally or fetched from a secret manager, *before other code runs*.

### The Problem(s)

**Problem 1: Mid-Execution Failures**

Invalid credential errors often don't reveal themselves until *after* your LLM has burned through a bunch of tokens. Or a temp file was created but failed to clear. Or you discover the directory path was wrong after it's already littered your system with garbage.

Try/except blocks inline with your functional code catch these eventually, but by then you've got the loose ends of partial execution to clean up. Headaches dealing with these results in people creating more and more and eventually redundant try/catch blocks, eventually finding themselves debugging 24 lines of functional code peppered between 500 lines of validation and error catching.

**Problem 2: Semi-Sensitive Information Aggregation**

AI makes it much easier than ever to aggregate semi-sensitive information into something actually dangerous. Not necessarily by doing a better job on a single filesystem, but by casting a MUCH wider net.  When I say semi-sensitive information, i'm talking about information which is not a concern in small volumes but, if scraped and aggregated, can add up to a security risk. A single file path visible in your code? Whatever. But enough of them and I can map your entire system. Write a testing suite to solve Problem 1 and you're likely just aggregating all of your semi-sensitive information into a predictable, easily scrapable location. This makes it much easier to hit the data volume threshold that pushes your semi-sensitive information into a full blown security risk.

**Problem 3: Environment Variable Hell**

The dawn of in-house AI has led to massive environment consolidation. Despite containers being trivially easy nowadays to call, and adding only microseconds of latency (nothing compared to waiting for an LLM response), it seems to be a growing design pattern.

Multi-agent LLM systems are a great example. You want to track each agent's usage separately, so you need separate API keys. All crammed into one environment. Now you're dealing with:
- `OPENAI_CUSTOMER_SERVICE_AGENT_API_KEY_v7`
- `OPENAI_DATA_ANALYSIS_AGENT_API_KEY_v3`  
- `ANTHROPIC_RESEARCH_AGENT_API_KEY_PROD_3`
  
Try to obfuscate your semi-sensitive information and you'll further increase this bloat with more package-level names sharing space in the environment-level variables.

And that's just the start. You get mixed up or make a typeo, and good luck debugging which of the 40+ similar-looking credential names you fat-fingered.


### The PyralGlo Solution: Validate Everything Before Execution Starts

PyralGlo replaces your mountain of environment variables, and all of the inline calls to your secret manager, with a single encrypted `HoG.enc` file per package, unlocked by one key in your environment.

**Front-loaded validation catches errors before execution starts:** Upon instantiation, PyralGlo decrypts your secrets/secret collection and immediately runs runtime tests. Every credential is present, valid, and unexpired before your package begins execution. No loose ends from partially executed code. No wasted tokens from LLM calls that error out mid-stream.

**Built-in tools to define validation logic:** Not just the variables themselves, but everything you need to verify them at runtime. Invalid credential errors become shallow and obvious instead of buried in your stack trace. Format-checking, expiration dates, test API calls, tools for all and more are baked allowing you to declaratively 

**Encryption protects the validation framework itself:** Front-loading all credential access and validation into a predictable structure could make it easier to scrape patterns and reconstruct your architecture. PyralGlo also encrypts the entire validation framework, so gathering everything in one place adds a layer of obfuscation to this information rather than surfacing it.

**Its so easy you'll encrypt even more:** The per-package credential access reduces environment clutter and bloated variable names.  This allows you to cast a wider net when deciding how much semi-sensitive data to obsure. File paths, config locations, internal URLs, the kind of information that's not critical on its own but becomes a security issue in aggregate. PyralGlo encrypts it all, so you can store what you need without leaking system architecture.

**Works with your existing secret manager:** PyralGlo is not aiming to replace existing secret managers.  It positions itself as a middleman to augment the security they provide with validation and convenience.

**LOCAL LOCAL LOCAL:** I don't want your secrets. PyralGlo doesn't keep your secrets with me.  PyralGlo keeps them with you.

### Key Rotation

**The Problem:** Encrypting all your credential acquisition in one place creates key-rotation friction. Botch the rotation and you've bricked many packages at once. This fear leads to long-lived encryption keys shared across multiple packages.  This is exactly the security anti-pattern you're trying to avoid.

**PyralGlo's solution uses its own validation-first philosophy.** Before rotating anything, the CLI tool verifies:
- All file paths are valid and readable with the old key
- Necessary read/write permissions exist for every affected file
- The new key is properly formatted and accessible

Only after these preflight checks pass does it execute the rotation atomically: update the environment variable/secret and re-encrypt all packages in one operation. No half-rotated keys. No broken packages. All or nothing.

#### Centralized Package Tracking

PyralGlo includes a vault-within-a-vault: the ability to create a special encrypted directory outside any individual package which tracks all PyralGlo-enabled packages in your environment. This registry itself is also encrypted (protecting your package list from reconnaissance), and enables:

  - **Atomic key rotation across packages:** Whether your packages share one encryption key or use different keys, the CLI can rotate them all in a single validated operation.

  - **Environment-wide credential updates:** Need to update a bootstrap credential referenced across multiple packages in the same environment? The centralized tracking enables distributing changes atomically with the same preflight guarantees.

#### Why This Matters

PyralGlo's core philosophy is package-level credential management, but existing key rotation tooling typically works at the environment-level. Encrypting your bootstrap-credential acquisition could stand in the way of healthy key rotation schedules, which would be a dealbreaker for security-conscious teams. 

That's why key rotation isn't optional or manual. It's baked into the tooling with the same validation-first approach which makes PyralGlo shine. The environment-level features exist to support the package-level workflow and have it work seamlessly with your existing stack.

For distributed systems spanning multiple environments or clusters, PyralGlo provides a binding for atomic operations to all packages in the environment. Your existing orchestration tools (Kubernetes operators, Ansible playbooks, etc.) can leverage PyralGlo's validated atomic updates as building blocks for coordinated changes across your infrastructure. Its built for bootstrap-credential rotation, but we have left the feature flexible and dynamic enough to do rolling updates of any PyralGlo defined variable. They will execute each step and stand at attention, proceeding to the following step only when given the programmatic go-ahead from your orchestrator, giving it a similar level of control to enable its own higher-level atomicity.

### Security

**PyralGlo is NOT a secret manager.** It doesn't try to be. Existing secret managers (HashiCorp Vault, AWS Secrets Manager, Azure Key Vault, etc.) are purpose-built for secure credential storage and are far more robust than anything PyralGlo offers. If these are your needs, make sure you use a dedicated secret management solution first before adding the PyralGlo layer.

**What PyralGlo actually does:** It's a credential validation layer that ensures all your secrets are valid *before* your code starts executing. The encryption exists out of necessity, not as a security feature. Aggregating all credential access patterns in one place could create an information leakage risk. PyralGlo functionality reverses this risk and adds a layer of obfuscation to these patterns rather than surfacing them.

#### The Encryption Layer: Defense Against Information Disclosure

PyralGlo encrypts your `HoG.enc` files to protect *organizational metadata*, not the credentials themselves:
- Which secret managers you use
- How you access bootstrap credentials
- Your system's authentication architecture
- Internal file paths and configuration patterns

This semi-sensitive information isn't a security hole on its own, but it can inform otherwise-capable targeted attacks. For example, if an attacker discovers you're using TPM-backed secret storage by seeing your configuration or acquisiton, they learn exactly which PCR registers, attestation policies, and sealing mechanisms are in play. This lets them craft precise attacks against your TPM hierarchy rather than blindly probing and risking detection without pay-off.


#### The Real Value: Obfuscation as Defense-in-Depth

For security-conscious organizations, hiding architectural details provides a small but meaningful barrier. It's not security-through-obscurity as a primary defense. It's obscurity as one layer in a defense-in-depth strategy. PyralGlo adds significant friction to reconissance without claiming to stop determined attackers.

#### Protection From Inside and Outside

Consolidating credentials in an encrypted package-level object has a free side-effect we'd also like to claim credit for: simplified granularity in developer access control.  Cluttering up functional code with credential management and error catching doesn't just make it less readable for inexperienced developers, it is also makes it difficult to give them access to the functional bits of code.  This friction often means they are either trusted to see all credential acquisition methods or they cannot work on that package whatsoever.  This can make it difficult to bring in new hands to help on certain projects.  You can give them access to only certain files, but this removes holistic context needed to be effective, and it shields juniors from the more systemic experiece it takes for them to grow in value.

By consolidating all of this into one folder, you can more confidently tune and grant package access to juniors and outside contractors, maximizing their ability to add value to the project without the increase in social engineering attack surface.

**What PyralGlo is good for:**
- Preventing mid-execution failures from invalid credentials
- Reducing try/except clutter in your functional code
- Avoiding accidental exposure of system architecture details

**What PyralGlo is NOT:**
- A replacement for proper secret management
- A security tool that lets you relax other precautions
- A safe way to commit credentials to public repositories (even encrypted)

The encryption is nice to have. You can breathe a little easier if you accidentally push `HoG.enc` to a private repo than if you commited an unencrypted `*.env`. But it's not a silver bullet. PyralGlo manages runtime credential validation while adding a *thin* layer of protection for configuration metadata. Use it alongside proper secret management practices, not instead of them.