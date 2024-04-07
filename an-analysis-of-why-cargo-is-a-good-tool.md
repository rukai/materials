# An analysis of why Cargo is a good tool

This article is written in response to an axo dot dev application question.
> Tell us about a tool you use often.
>
> We're interested in learning more about what you think makes a good tool;
> including what aspects you choose to analyze as well as how you analyze those aspects.

## Preface on analyzing tools

Thinking about it, I can see two competing axes in which one can evaluate the quality of a tool:

* Effectiveness - How much it reduces the time, complexity and risk involved in a specific task
* Diversity - How many different kinds of tasks it can facilitate

This would give something like `goodness = effectiveness * diversity`, this of course is kind of nonsense, reality doesn't work like that.
But it can help when thinking about how tools choose where to spend design points.

An important part of designing good tools is looking for opportunities where you can slice Effectiveness and Diversity nicely to get large values for both.
Even if there are many other existing tools in a domain area, if there is a unique slice of Effectiveness and Diversity that is lacking in tools it could be worth writing one!

For example:

* A custom shell script written to automate something on your machine and which takes no arguments has minimal diversity but is maximally effective (assuming no bugs or inefficiencies). Just run the script and its done.
* Tools internal to a company drastically reduce diversity to only meet the needs of users within the company but in return are much more effective at solving the problems of the employees.

## Cool, now tell us about cargo

Now lets analyze Cargo with this framework.

### Effectiveness/Diversity split

First off, how does Cargo slice Effectiveness vs Diversity?

Cargo is a language-specific build system, serving only rust developers.
Being language specific greatly increases Cargo's effectiveness, it reduces complexity for the user, streamlining project setup and maintenance.

Cargo further prioritizes its open source users first, internal company projects second and big tech company monorepos last.
While not explicitly stated, observing the project's priorities makes this clear.
For example, many companies were blocked on using rust due to the lack of private registry support.
Cargo now supports private registries but it took a long time to get there.
Also, Cargo's out of the box support for dealing with huge rust monorepos is quite poor.
To stop Cargo from constantly rebuilding a large chunk of the dependency tree when swapping between crates or enabled features, a tool like [cargo-hakari](https://crates.io/crates/cargo-hakari) is required.
And if the monorepo needs to be cross language, then Cargo is quite poor at interacting with other languages.
In such a case I've heard that companies are better off with buck or similar.

This reduced diversity enables Cargo to reduce its exposed configuration interface:
Cargo's build configuration follows this sequence of fallbacks: "sensible defaults" -> "toml configuration" -> "build.rs"

#### sensible defaults

When possible, the best interface is no interface at all!

Crate source code is read from `src/main.rs`/`src/lib.rs`.
Examples are automatically discovered in `examples` and integration tests are discovered in `tests`.

Cargo completely hides concepts like intermediate object files from the user.
Linking is mostly hidden with a few configurations available to use alternate linkers and rustflags.

#### toml configuration

Cargo packages are each defined with their own `Cargo.toml` file.
`Cargo.toml` includes basic package metadata such as name, version and license.
But also includes a list of dependencies to other packages, including the dependencies version and where to fetch it from.
Pretty much all crates need to configure these, so driving them from a config file allows Cargo to hide a lot of underlying complexity from the user.

Additional user and project level configuration can be set in the `.cargo/config.toml` file.
Having some project level configuration in the separate `.cargo/config.toml` instead of in the `Cargo.toml` makes discoverablility worse.
But having config that can be set in both `.cargo/conifg.toml` and `Cargo.toml` isn't ideal either.

#### build.rs

Now what if the user needs more powerful configuration than possible by writing configuration files?
Usually this is where DSLs get involved, but Cargo instead introduces the `build.rs` file.

The `build.rs` system nails the effectiveness/diversity split in a way that other build systems often fail at.
Most build systems will require users to master (or, more realistically, cobble together a solution with) some DSL or turing tarpit configuration file.
Instead Cargo gives you full control over the process in a real language, rust, the very same language you are already using to write the project itself.
This sounds dangerous, and indeed currently has many issues with reproducibility and sandboxing.
However I believe it's still worth it.

`build.rs` covers many use cases including:

* Generate rust code into `src/`
* Set env vars later included into the project via `env!`
* Build and link to external C dependencies

`build.rs`'s success relies on access to some of Cargo's other killer features, semantically versioned crate resolution and crates.io registry.
Without these two features, all build.rs scripts would be either writing all their functionality from scratch or hitting endless dependency issues.
With the knowledge that the build.rs has robust access to dependencies, large chunks of useful compile time logic can be shared between crates.

<!--
#### Missing configuration

There is no `package.rs`

Not being able to target cross language and huge monorepo use cases well sounds like we are leaving many users in the dark.
But there are better tools for that! buck etc.
Also those huge companies can just fund their own tools to fix their own scale issues if needed.
-->

### Further analysis

So far we've looked at how Cargo chooses its feature set to maximize Effectiveness and Diversity.
But theres so much that goes into a tool design.
This is just a list rather than a cohesive analysis but heres how Cargo fares at some other important aspects.

* Implementation quality
  * Cargo commands complete quickly enough, I've not noticed them ever being the bottleneck when working with Cargo.
  * Cargo's implementation of stable features is very robust. I think I've encountered just one [implementation bug](https://github.com/rust-lang/cargo/issues/4941) in all my time using Cargo. Or at least only one that annoyed me enough into reporting it.
* CLI design
  * Cargo's CLI is split up into subcommands `new`, `build`, `run`. etc
    * Splitting up commands rather than a giant soup of arguments is clearly preferable.
       It might be tempting to combine `build` and `run` since there is some overlap in the args, however splitting them up is the right call due to the numerous args specific to either `build` or `run`
  * Cargo's CLI API has benefitted greatly from the presence of [Custom Commands](https://doc.rust-lang.org/book/ch14-05-extending-cargo.html)
    * If Cargo is lacking some kinds of helper functionality, custom commands can immediately fill the need for users, without the project feeling the need to rush into a solution. From there Cargo can examine the popular custom commands and integrate the ones that the project is ready to commit too. See for example [cargo add](https://github.com/killercup/cargo-edit?tab=readme-ov-file#cargo-add) which has now been integrated into Cargo.
* Familiarity
  * Cargo would be familiar to developers coming from build systems like npm, but for developers of other origins they will have a new paradigm to learn. Developers who haven't worked with command line tools before have even more to learn.
* Documentation
  * Cargo has [reference level documentation](https://doc.rust-lang.org/cargo/reference/index.html) sufficient for day to day usage. But I have run into poor documentation for obscure things before.
  * Additionally Cargo has a great many tutorials: [the cargo guide](https://doc.rust-lang.org/cargo/guide/index.html), the rust book itself, and being the defacto build system for rust most tutorials on the web will be done through Cargo.
* Modifiability
  * Cargo is open sourced under MIT+APACHE, very permissable licenses.
    If a developer fixes a bug or adds a feature they need, they can:
    * Contribute it upstream for everyone to make use of.
      Unfortunately the Cargo team is majorly understaffed and it can be quite difficult to get PRs through.
      Adding new features goes through RFC's which is most likely not going to be accepted purely due to resource constraints.
      But even trying to fix bugs in existing unstable features is very difficult.
      Not ragging on Cargo developers here, this is purely a funding issue from big tech companies.
    * Maintain a public fork. This is widely considered a bad idea, an incompatible fork could fracture the ecosystem. I do however value that it is possible.
    * Maintain a private fork. Where a company has control over the tools used by developers they can maintain a private fork of Cargo if needed.
  * Cargo is written in rust, so any sufficiently experienced rust developer can dive into the codebase themself.
* Obtainability
  * Cargo is just there, waiting for you.
    * Ok, you have to install rustup first, but if the user is using rust through the recommended installation method they need to expend 0 effort to also access Cargo.
  * In the corporate world, a common issue with obtainability could be downloading executables through a corporate network. Or an inability to run executables and/or admin rights on a corporate managed machine.
    * However at least Cargo (and rust) is open source and free, there is no licensing or costs to get past management.
  * Sometimes a specific version of Cargo (tied to the rust version) could be needed for a project. Assuming rustup again, the project can pin the rust version used by the project via their `rust-toolchain.toml` file
* Compatibility
  * Cargo supports all the major OS and CPU architecture combinations.
  * If a cargo project has a non-cargo dependency it can be handled through the `build.rs`.
  * If a non-cargo project has a cargo dependency they either need to:
    * Invoke `cargo build` and then read the output files from the `target` directory.
    * Include some kind of cargo compatibility or conversion for example: [reindeer](https://github.com/facebookincubator/reindeer) does this for buck
  * Cargo provides a [Custom Commands](https://doc.rust-lang.org/book/ch14-05-extending-cargo.html) system, this provides a point for tools to integrate into the cargo workflow, but with only the most minimal of interface commitment from Cargo. If a custom command needs access to Cargo features they should invoke Cargo from a command like everyone else does.
* Accessibility
  * CLI tools generally perform well with tools like screenreaders, I don't use a screenreader myself but I cant imagine anything in Cargo that would inhibit use with a screenreader
  * I have used Cargo with voice dictation software, talon in particular, and its fine, no different to other CLI tools really.
<!--* Configurability
  * Allows a tool to expand its effectiveness by providing user/project level configuration to better suit the users/projects needs.-->
