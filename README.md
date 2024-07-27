# Software Engineering Manifesto

During the course of my career I have been lucky enough to use a number of different tech stacks and programming languages, and to be in the presence of some very talented engineers while working at organizations that had a high commitment to engineering excellence. 

I have also led engineering and data science teams, owned major product initiatives, and architected significant pieces of complex real-time high-throughput high-uptime software systems. In addition, I have a longstanding focus on algorithms, machine learning, and intellectual property. 

The following engineering manifesto distills much of what I have learned along the way. I’ve learned to be practical - attachment to a particular piece of tech due to familiarity, industry hype, or shiny newness are generally counterproductive to making the best choices for your engineers and the company. As it turns out though, most of the core principles that I believe make for a successful and enjoyable software organization, are not tied to a specific piece of tech - they're really rather universal. 

## Code of Conduct

It all starts here. Every organization needs an ethos - the guiding spirit by which they operate. In a software organization, I believe that setting the right tone, pace, and commitment to excellence creates the essential foundation upon which all other guidelines grow.

An ideal software organization is one in which: 
- engineers lift each other up
- engineers feel like their voice matters
- engineers feel like the work they do is important
- the pace of development and delivery springs forth from the engineers
- the drive to create excellence arises from the engineers

The best software gets delivered when engineers are encouraged, empowered, and supported in becoming their best, most powerful selves. When that environment is present, the drive to create, the drive to deliver excellence springs forth from within. When a team has this, you can tell immediately. 

The leadership team can cultivate this. There is the classic adage "reward the behavior that you want to see more of". And that's very true. If someone goes above and beyond, showing that hustle to resolve a production issue, mentor another team member, or deliver a particularly awesome piece of code - timely, specific recognition is crucial. It's more than that though. It's about taking the time, as a manager and/or project lead, to maintain a sense of where your team members are capability-wise, and then tee up project pieces that are going to be right on that edge, where it's going to push them to grow, but you are pretty sure they will succeed, and then they get that rush, that joy at delivering something they have every right to be proud of. 

Even if you are no longer officially doing code reviews for your IC direct reports, take the time to scope on a few of them every so often. If you manage data scientists, have them present their latest result to the team, and make sure you attend. Few things are as encouraging to a team member when their manager talks about something they are currently working on, and compares it from memory to something they did six months ago, and points out how they've grown. It reinforces that they are a real person and that the manager cares about their growth. 

When managers give positive feedback, teams will emulate this. When managers solicit opinions from ICs, and listen, team members will feel like their voice matters - and everyone likes to feel included. 

Take the time to pass positive customer feedback to the engineers. Let them know how much their efforts are helping make somebody's life beter. 

Creating this kind of environment in turn creates a virtuous cycle where engineers will drive themselves to deliver at greater levels of excellence.

Once that foundation is set, then you can focus on the following...

## Code Quality

### Code Principles & Patterns
1. Use Dependency Injection
    
    As programmers, conditional statements are one of the first things we learn. Which is ironic because use of them to control any type of logic flow based on multiple terms is, along with mutation, probably the number one source of missing test coverage, and therefore bugs. That might sound controversial, but it's actually straightforward. There are two driving reasons for this. 
    
    The first is that it's virtually guaranteed that if you encounter an if statement with three terms that it does not have full test coverage. Why? That's eight tests! No programmer wants to write those, so you get two to three with the common cases, where the coder is relying on assumptions about what they expect to see evaluated in the conditional. And that's where the bugs, whether immediate or in the future, hang out.

    The second is, it's very likely that if you have to have conditionals around multiple terms in one file, especially in any sort of event processing system, you are going to have either exactly the same or slight variations on it in other files. Now you have to replicate those sets of conditional tests in multiple places. Which no one wants to do either. So it gets skipped and then what happens is later, someone makes a change, searches for and updates existing tests because they assume everything's covered, and then that lagging case in that other file gets missed. And that's where bugs hang out.
  
    But there is an easy solution: dependency injection.

    Particularly in web applications, a request hits the web server stack, gets routed to some handling logic, and from there gets processed by the business logic. Typically, there is an opportunity to create a class instance at the start of the business logic, like way up in the view, and use dependency injection to pass that down through other business logic layers. That is the right way to do it, almost always. There are frameworks out there, like FastAPI (which I like quite a lot), that tend to encourage just trying to do all of the API handling logic in the handler method. While I understand the ethos, I've seen that results in bugs and the code getting very non-DRY. There is something so much better when you see a FastAPI view file with a set of methods each of which is 5 lines long - basic arg handling, class instantiation, toss the class to other functions or classes, then return the resutls. It's jsut so much cleaner. 
    
    Use dependency injection to avoid stacked conditionals.

2. Use SRP (Separation of Responsibility Principle)

    Clean contracts and clean definition of what exactly a given set of code is supposed to do is crucial to developer happiness and velocity. Most (maybe all) developer velocity formula calculations don't have a way to account for the impact of SRP violations in the code base tripping them up. But you can see it clearly when you lead a team.

    SRP violation signifiers come in two related forms. The first is, how much of the overall system does the programmer have to understand before they can change one specific part. If the architecture has grown without SRP enforcement, then the answer will be "way too much". The second place you can see it is, perhaps surprisingly, in how fast programmers can write test setups for their code.

    When tests get complex, it’s often a reflection of poor architecture - one or more elements did not get a clean encapsulating contract around it, and as a result, test setup alone requires x different objects being assigned specific values to create the right context for the test. This is a sign that organizationally, you have missed an opportunity to either design initially, or slice off appropriately, a good piece of business logic into a library or a service. If the code base is using an ORM, you will almost always see Law of Demeter violations in concert with really complex test setups. Yes, it hurts to fix this, but at the least, if you are not tracking the need to separate things, then you are fooling yourself about a very real team-velocity drag that follows from the SRP violations in the code base.

    Reinforce a SRP mindset among the engineers.

3. Law of Demeter

    LoD tries to capture the notion of how many interrelated classes/instances does one have to jump across to get to the data they need. Typically in a web application, this can show up through multiple has_a/has_one relationships. On their own, those are useful and fine, they're a great tool. But, if the class architecture has either gone too monolithic and doesn't have good boundaries, then you can have instance links ten terms long. 

    And that's not good, because, among other things, it means if you ever want to try and split up that architecture, you are in a world of hurt. Good luck slicing all of those foreign keys out. 

    There are alternatives. The easiest is to simply provide reasonable methods on the more top-level object to retrieve instances that are under their logical purview, but where the accessing/calling code really doesn't need to know the chain of interaction to get to the piece of data it cares about. Ruby has a nice "delegate" operator that can help with this, but it's more about designing with the thought of how deep into the details of a certain area of code should other areas of code really be able to see?

    Another alternative that can be quite useful, especially when you have multiple data stores throughout the system, is better use of unique identifier strings. Guaranteed unique uuid strings can be a wonderful thing. Sure, if all the data is just in one relational database, then you will likely be adding some overhead and you're going to have to think more about things like how handle deletions (since you can't just rely on an ACID cascade). But the tradeoff is very often worth it, especially in any kind of event-driven system that has to handle/process/store lots of event data, since in such situations you likely need and want to not just jam all of that in one SQL database.

    Use the Law of Demeter to guide your SRP architecture.

Those are my top three guidelines for creating good code. 

### The Code Review Process

Ideally we create and add to code bases that we are proud of. When the original code and foundational structure is right, it creates a virtuous cycle where further commits need to meet the standard that has been set. It is imperative that new engineering hires be properly educated and onboarded to the coding standards of the organization. 

Particularly if too many engineers are brought on at once, code quality can take a dive. Once that happens, and the code base gets infected with low quality code, it’s nearly impossible to recover from. Pull requests are, of course, designed to prevent this from happening, but if the engineers doing the review are also new to the org, then things can fall apart very quickly. 

The particulars for your org may differ, but as a rule of thumb, plan on an engineer being part of your org at least 3 months, and maybe up to 6, before they are given the green light to perform solo code reviews. If your organization is still quite small (15 or less engineers) then this timeline may be much shorter, as typically there is more direct contact between the overall owner of the code base and the other engineers.

Aside from following the code standards of the organization, on a meta-level there are three key criteria that the reviewer needs to follow for a successful code review:

1. Timeliness: Set a target for code review turnaround. 48 hours is probably a bit long. The really good teams can usually turn these around in less than 24 hours, and usually have a target of, for instance, if a PR got issued in the morning, it got reviewed before the afternoon was out. It’s a cadence issue. Effective engineering teams operate with a cadence. Moving the ball fast down the field together is key. Keeping the PR’s small with fast turnaround really facilitates this.

2. Exactness: Ask clear questions, make clear suggestions, and if you see something you like, give clear props.

3. Simplicity: There’s probably some debate here, but there are various PR guidelines out there that include sheets of 20 emojis for the code reviewer to use, as a way to classify a comment. This is completely unnecessary and distracting - it’s like someone solved a problem nobody actually had just because they wanted to be cool. Keep it simple.

If your engineering organization is really cooking (everyone is in a good flow state), then pull requests should be coming through often. It’s important to not overly burden engineers with the book-keeping part of the process. PR templates are fine, but keep them to 3 things to fill in. Otherwise, what happens is, engineers get tired of filling out the 10-20 fields that someone thought should go on the PR template and they just ignore them - so you end up with pull requests with template scaffolding mixed in with the important bits. Coming back to the idea that simplicity, cadence, and effectiveness all work together - keep the overhead low.


### The Pre-Code Review Process

1. Naming Conventions
    
    Most languages have syntax conventions for names (camel case, underscores etc), and this gets particulalry enforced if you are using an ORM layer to assist with any sort of autogeneration, but the semantic nature of how variables and functions/methods get named can vary quite a lot by programmer. It's easy to enforce syntax guidelines with a linter, but semantics are a bit trickier to enforce automatically. It is generally easy to spot in code reviews, however, as long as guidelines have been made clear. Simply having clear guidelines is the most important step, regardless of the specifics.

2. Extraction of scripts and setup into easily repeatable single line commands
    
    There is a famous xkcd comic that tried to make the point that spending too much time on automating a task that is not too onerous or done very often is not worth it. While there is some truth to that viewpoint, that comic was probably grossly responsible for many folks thinking about it the wrong way. That’s because it did not include factors such as: 
        - How many other people in the org will need to perform this operation? 
        - What is the time cost for folks new to the operation vs the person who has done it many times?
        - What is the cleanup cost if someone gets it wrong?
        - Will it need to be run in multiple environments?
        - Does it need to be part of a CI/CD pipeline?
        - If there is a major production fire, will this script be needed to diagnose and/or fix?
    These are all very important to consider. So the problem with the xxkcd comic is that is focused on "you" instead of "team", and it didn't take into account the fact that one's ability to properly estimate the scenarios for its usage is necessarily limited. Therefore, having the original engineer on a piece of setup/code/deployment/etc create one helper script for others to use can save serious amounts of time and headache for other team members down the line. 

    One good rule of thumb here is, if you have shell commands that require more than one line, or are longer than 80 characters, put it in a .sh or .py file. Simple, easy.
    
    If you find yourself doing the same configuration action twice, in particular if you are doing so because you got something wrong the first time, put it in a script as you go. Without it, others will likely make the exact same mistake you just made.

3. Lint prior to code checkin
    If code reviews are just enforcing basic code style then they are a waste of everyone’s time. Linters are better at this. There’s an 80/20 balance in the linting rules, however, and in that 20, there tends to be more and more cause for reasonable situational variance. You don’t want the linter making noise on those because then people start ignoring it or hating it. Linters work best when they are configured to act as a friendly pair programmer, not a squawking bird. 
    
    I personally use flake8, black, djLint, isort, and pre-commit together.

4. Git hooks to enforce that only great code makes it to the pull request phase
    Git hooks that run linters are wonderful things. 

## AI Tools

There are a number of LLMs and associated tools to help auto-generate code. While there is a lot of promise to these tools, they also have a long way to go. If you know what you're looking for, and you have a pretty good sense of likely right answer vs wrong answer, then these tools can be very effective. For more junior programmers, or folks operating in a new area, the tendency of these tools to produce half-right solutions can be very problematic. 

My personal prefence is this:
1. In VS Code, I run both Github Copilot and Google Gemini.
2. Separately, I have a repo of python helper scripts that call several of the LLM APIs. I have iTerm2 configured to take voice commands to generate code, and the request fires up four iTerm windows with different results. This works very well because then I can compare the four answers. Often times the right thing to do is extract part of one answer, part of another, and use those to help create the final piece of code.
3. I also have Claude Sonnet and Perplexity configured for me to ask coding questions in the languages and environments that I use.

This type of approach has the benefit of forcing me to reason about the pro's and con's of multiple takes on a problem. The biggest win comes from getting answers that are more you-know-it-or-you-don't and the alternative is like, a deep dive through obtuse documentation to get a single line answer. 

Like any set of tools - use with caution and reason. 

## Test Coverage

1. Strike a balance

    I’m a big fan of test coverage. I wrote the E2E test system at my last company, which was interestingly complex because it actually issued and processed real phone calls (which was at the core of our processing chain). So you know, I’m all in. That said, when test coverage gets excessively complex, either in the setup or in the number of combinations being tested, it's usually an indicator of other problems or inefficiencies in the code base (almost always, violations of the Code Principles & Patterns list above). The best approach here is to empower your devs to refactor when they see this - give them space on a project to create refactor tickets when they see the test hairball start to grow. The argument against such refactoring will almost always be "well, the business logic itself is complicated". That is rarely a good argument. Logic is logical and the different aspects of it can almost always be separated out. 
  
2. Unit test coverage

    Simple, repeatable. Encapsulate related business logic into its own class with clear tests. Try to keep the classes at no more than 200 lines of code. This keeps the test file to a size that a developer can reasonably reason about. If a unit test file is more than 400 lines of code, something has gone wrong in the class structure architecture. Of particular note here - keep your business logic out of your ORM database object classes! It is so tempting to mix them together. Don't do it - keep that ORM class focused on the object lifecycle - enforcing validations, etc. Keep the business logic elsewhere.

3. Clear strategy for test data

    Factories can be great, but they can also be dangerous. A common pitfall: factory generated instances that are actually invalid but the engineer hasn’t tried saving the instance to the database or calling is_vald? or similar on it. So the test passed, but the instance fed to the test will never appear in actual use. 
    
    Seed data can be great, but it can also be too heavyweight. Developers need to be enabled to roll with good cadence. In a big repo, waiting for long-running sample data generation scripts is a drag, and the larger the team gets, big sample data is a recipe for merge conflicts. 

    A middle ground between factories and seed data is to use both, but at different times in the lifecycle of a given part of the codebase. If some set of classes, say for a given service, are mature and not likely to change much, create a tech debt ticket to transition factory test setup data into seed data. If an area is still under hot development - stick to factories.

4. Pay close attention to your mocks and stubs

    We all love stubbing and mocking in our tests, right? Well, ok, maybe it’s a mixed bag. Certainly it can feel good, and if there are good pinch points where we can do it easily, that’s a good sign that our SRP is going well. That said, you can also fool yourself with it. One example of this is in a system like Rails or Django where the test framework is mocking certain things like the specifics of the request as it comes in, including http verb, routing logic, environment inclusion, and any intermediate processing that may be occurring in the processing stack prior to the request hitting the processing code. That’s where it’s good to have at least a few E2E tests that, yes, are likely brittle, but allow you to safeguard that nothing has gone unduly wrong across contract boundaries.

5. TDD - when appropriate

    This may be controversial to some! TDD is great, to a point. When the JIRA ticket is well-defined, with at least decent specifications of the go-right case, then TDD can be fantastic because writing the test statements first identifies anywhere the expected behavior is not clear. So the programmer can go ask the PM - what do you want in these corner cases? But, particularly when it's an exploratory feature, and especially if it's for front end pieces that may change, TDD can be putting the cart ahead of the horse. Sometimes its ok to say, we're not doing TDD, but let's make sure the tests are in before this branch gets merged to main.

## SQL Database Table Structure & Usage

1. Clear guidelines
    
    If you’re using an ORM layer like with Rails or Django then much of the database syntax standardization is going to come from the migration generation logic these systems provide. However, if you are using something a little different, say PostGIS, or you are using a system like Flyte, then it’s worth it to have some syntax guidelines in place - in those cases there’s a syntax that comes standard and it’s worth making sure your developers know what that is. 

2. Separate RDBMS's
    
    If you have a multi-part system it can be tempting to shove everything in the same RDBMS. After all, each RDBMS can support x number of databases within it, and it’s more work to set up a local environment and a cloud provider with multiple RDBMS’s, so - why not just shove everything into one RDBMS? Hold on. This is the difference between theory and actually having to support, and pay for, production systems. 
        - First, for starters, big RDBMS’s in production are expensive single points of failure. As you try to scale them vertically, the cost metrics get insane. 
        - Two, taking such an approach limits your options for swapping/upgrading parts of your processing. For example, let’s consider an ML pipeline technology like Flyte. It’s k8s with a Postgres instance designed to take a helm blueprint and perform complex processing. You can set this up on a cloud provider like AWS or GCP, and it works. Now, you can also do that sort of thing with a provider like DataBricks. And DataBricks is a great system with a lot of features that make life very nice for data scientists. If you keep your Flyte pipeline, k8s structure and Postgres database completely separated from your front end app, then if you wanted to move from Flyte to DataBricks, or use both, you could do so very easily. Like, literally, no one working in the web application will even notice. Not so if you end up having to perform database cleanup on a central RDBMS. (okay, so yes you could partition etc to ease this - but still). Another example would be say, using PostGis, which needs to load extensions into the RDBMS. Here the versioning and global impact in the RDBMS really show the wisdom of simply keeping a separate one for PostGis operations. 
        - Three, keep big background processing tasks from affecting your front end user experience. 
        - Four, prevent any temptation to cross SRP boundaries. 
        - Five, in the example given, these databases are used in totally different ways, will have totally different needs, and things like migration, backup, and restore strategies are going to be very different. Don’t try to fit that all under one roof.

3. Database health check

    Use psql and explain to check query health. Have this as a recurring tech debt ticket so that eventually every programmer in the org has spent time doing this. 

## REST vs GraphQL

1. Use REST when it is sufficient. Use GraphQL when it’s necessary.
    REST handles most API duties just fine. Sure, GraphQL was the hotness for a few years, and I’ve written my share of it, but it’s often overkill. And I'm speaking as someone who is really into graph data structures! If your product is an API into a large collection of data, and you really need your clients to be able to efficiently query specific fields from a DAG or Graph style relationship, then GraphQL is the right choice for that. But let's be honest, a great deal of the GraphQL out there is just used as a semi-convenient way to do API contract definition between a front end mechanism like react and a backend mechanism like Rails - driven by libraries like Apollo (for React) that promise to auto-update all the things in the UI.  
2. Rest or GraphQL: use generator tools/scripts
    If you have a React (or js framework x) frontend talking to a Django or Rails backend, make sure the entire request chain method definitions can be generated by a tool or script. The reason is, developers’ brains gets scrambled trying to jump seamlessly between the different syntaxes, and it is highly likely that forcing devs to do so will result in mismatched definitions (plain ol syntax mistakes), and these can be really difficult to track down because often, the error messages, especially on the javascript side, are terrible and misleading. Do your devs a favor and get this right.
        - For Rest, OpenAPI is a good choice.
        - For GraphQL, roll your own wrapper around the graphQL generator.

## DRYness

1. Code

    Always

2. Data
    
    In a normalized SQL database driving a web application: pretty much always.
    
    Sometimes though, depending on the tech and the performance requirements there can be really good performance reasons to de-normalize. A good example of this is using a data lake, like the Delta Lake that comes with DataBricks. It has a particular processing paradigm, which is good for data science type queries, where the idea is to pull large sets of information from a table, perform parallel spark based processing on the results, and then return the final transformed/distilled/calculated values. Due to the way the underlying parquet file structure works, you can only have up to three indexes on a given table, which means that your options for doing relational “style” queries is not very great. Thus, a different in set of techniques comes into play, including lightweight temp tables, the distributed spark processing, and sometimes, having the same data in more than one place. Now, as a matter of sanity, but also practical usage, if data is in more than one place it really needs to be immutable. Which is typically fine - this use case is commonly for “data of record”, ie events that have occurred whose key attributes shall not change. 

## SRP revisited

1. Repo level
    - Separate the environment setup from any one repository
        - This might seem like a minor one, but conceptually, it sets a tone of SRP that is actually really refreshing, especially if you’ve ever experienced working with (or tried to slice up) a monolith web application. Take a web app framework with an ORM, like Rails or Django. You start with one repo and after you get your initial data base setup and configure the connection info for the ORM, you are going to start generating migrations. Because you create the migrations, and apply the migrations, from this repository, it may seem natural to put all the local database setup information in that repo. What happens though is that it sets a tone that the application owns the database, and scripts and other items, that are also typically environment dependent, start to show up in the web app repo that should really conceptually be none of the web app’s business. When that app gets deployed to AWS for instance, that database is over in RDS land, and really, all the app should know is that (with Django for instance), that the default database connection routes *somewhere*. Now, if you are using more than one database, like, let’s suppose you are also using a PostGis database, it will start to feel over-stuffed quickly, if all the setup, readme, and associated material is all in the web app repo. So, separate it at the start and make life better. 
    - Separate the react app from the web server
        - I am assuming a situation with Rails or Django serving a front end that uses react, possibly as one react app, or maybe several react apps, but there’s no npm server running - it’s Django etc serving the page, handling routing and authentication, and react got started via an included built/bundled javascript file. I prefer this sort of setup to web apps that are all react and node.js because it lets the pieces focus on what they are good at. 
        - Now, some folks who take this approach will want to drop in the react code into subdirs in either the monolith app or similar. Eventually, they will be forced to pull it out, so may as well start with it separated. The reason is, as the engineering team grows, you will go from engineers who cover all the bases to folks getting a little more specialized, and your front end devs probably don’t need to be caught up with everything going on in the web application server. Let them roll lightweight in the environment they are used too. But, create a seamless and fast way to kick out new javascript and try it out. Here, I’m a fan of esbuild - it is lightning fast at building javascript. 

## Separating Background LLM Processing Pipelines

I call this out specifically because LLM processing, or more generally, training or fine-tuning of ML models is any area in which available services, costs, and available tech is changing so fast that today's leading model and most cost-effective fine-tuning option may be old news next month. Thus, flexibility and good abstractions are key. This means keeping that part of your application completely spearate from the other parts. That doesn't mean that the same infrastrcture can't also have an inference API available to your main application, but the point is, the main app should have no idea that it's being served by the same infrastructure - it just knows it's hitting an API. 

Typically, with ML pipeline training or fine-tuning or research, there's a variety of ways to get the data processed. Perhaps suprisingly, and this is coming from someone who has architected and created features with Kafka, RabbitMQ, and similar message passing systems, sometimes the best choice is a shared S3 bucket. Which just sounds so primitive. But, there is a very good reason for this. It turns out that for a system like DataBricks, it actually excels at importing data from S3 buckets. Say what? Yeah. Their importer is optimized for writing data into Delta Lake parquet files, and the manner in which the data gets ingested makes a huge difference to how well those parquet files are formed. So they have a very well thought out S3 autoloader that keeps track of all the files in the bucket and it knows which ones it has imported or not. 

That also creates the advantage that you can, with minimal setup, mirrror (auto-replicate) that S3 bucket in a separate AWS region. The advantage here is that, if anything goes down, your data is just chilling on S3, and when DataBricks starts up again, it will continue where it left off. 

Thus it's important to do some pre-benchmarking when considering these kinds of options, and sometimes the simple route is in fact the best.

## Wrapping It Up

The contents of this manifesto are all from lessons learned along the way. I feel lucky that I've been in the mix for high-throughput, high-availability, real-time software systems, where we were running multi-region, multi-cloud, multi-country, and working on combinations of standard features and data science R&D. I've worked with some fantastic engineers and I hope that I've brought out the best in some of them, just as others brought out the best in me. When we can code in that flow state, because the right guidelines have been put in place - just enough to keep us on the road, but also support us in having fun and moving fast - then programming is a joy. 

