Build and test my project:
    This project uses maven for dependencies, build, and testing. Mvn install
    at the root module should compile all modules and execute all tests.

Run instructions:
    This project depends on a RabbitMQ host that is configured to support delayed
    messaging. The instructions for enabling delayed messaging can be found here:
    https://www.rabbitmq.com/blog/2015/04/16/scheduling-messages-with-rabbitmq/

    There are two modules that must be running: WebServer and TaskConsumer. Each
    project has a <module>EntryPoint containing the main program. Configuration
    is managed by editting the config.json files in the respective resource
    directories of the respective modules. The default configuration points at
    localhost for such a Rabbit server.

Tech dependencies and considerations:
    I chose Java for this project because of Maven. This project clearly involves
    the use of lots of standard and open libraries; fetching website content,
    sending emails, integrating with RabbitMQ, JSON encoding tasks, exposing
    an HTTP interface for registering tasks, etc. Maven is a fantastic build and
    dependency management system, so the simplicity of using so many external
    libraries was great.

    I would like to call out that I have very little professional java experience,
    so if I have strange naming conventions or I'm missing some common Java pattern,
    my apologies. I became accustomed with Maven while working on the old server stack
    for my last project in scala.

    This was clearly a producer/consumer problem; with an intermediary exchange
    we're able to scale the number of HTTP servers registering tasks and the
    number of consumers polling websites independently. RabbitMQ is proven to
    be a highly scalable technology and somewhat of an industry standard so I
    chose to use this. One major concern I had was scheduling; a bit of searching
    told me that there was a delayed messaging extension for RabbitMQ (which I used,)
    however it seems to have troubles supporting some failure cases. As I point
    out in comments, there's no atomic operation with rabbit to queue a new message
    and dequeue the current one. I want some kind of task swap operation. The other
    option I considered was Quartz, as it's a proper scheduler and you can have
    multiple consumers of jobs across multiple processes. I have a bit less trust
    in Quartz, and I believed Rabbit would be easier to set up.

    This also depends on the google mail service; I didn't want to require us to
    be running our own SMTP server locally.

    I wanted the task objects to contain all state for processing an iteration
    of the polling, so the consumer could be stateless. I wanted them to remain small
    and hence I stored the hash of the website content instead of the entire page.
    I also assumed it was valid to watch just the requested url/page, and that this
    problem did not involve crawling the links on the page to cover the whole site/domain.
    I also wanted to JSON encode them; there's probably a default serializer that
    would have worked, but I wanted the tasks to be operable in multiple languages
    and with some flexibility concerning versions of the process. With some graceful
    handling you could change the format, and have old and new consumers/producers running.
    It's not something you'd want to do but it'd help with zero down time deploys
    when you update the code.

    One other thought I had was that with the processes being small and stateless,
    you can easily let them crash. If there's an error connecting to the RabbitMQ
    host or the email SMTP server, you don't really need to handle the error at all;
    let it crash the process and use some process monitoring to relaunch the instance.
    No need to write reconnect on failure code...

Scalability and future extensions:
    With RabbitMQ as an intermediate exchange broker, we can run as many WebServer
    instances as needed, and as many TaskConsumer instances as needed. The RabbitMQ
    delayed messaging feature does not post the message to the queue until the delay
    has elapsed; this means that any message in the queue is work ready to be processed.
    Monitoring this queue would therefore immediately show the need for more processing
    power. If the queue started growing, you would add more TaskConsumer instances.

    One performance concern I had was multiple users monitoring the same website. I'm
    scheduling a task and polling the website once every 5 minutes for each subscriber.
    If a million people subscribe to changes on a site, that's me polling a million times
    every 5 minutes, instead of once. I considered a list of listeners for a given site;
    without a way to find the existing task I would have had to store that set in a
    database that I could look up once I've detected a change. That presented two problems:
    an additional external service dependency (the database) and the fanout problem once
    you've detected a change. If there's actually a million listeners, you don't want
    one TaskConsumer iterating that set trying to send all the emails. What happens if
    you send 500k and then crash? Without the job being updated in RabbitMQ, another node
    will pick it up and mail those first 500k all over again... In a sense, it's not
    efficient to poll once for each listener, but it is scalable.

    There was a nagging consideration that's been bothering me for things not in the
    spec as well; eventually someone will want to cancel a watch. There's no easy way
    to look up their task and delete it. If I had task IDs I could maintain a set of
    cancelled_tasks, probably as the primary key of a table. I could look up if the
    task was cancelled while processing it, and if so drop it from the queue. It's
    another remote call (a DB query) on every task processing, but it'd work...

    The other consideration is persistence, however you could use durable queues
    and that might just take care of itself... although I do wonder how delayed exchanges
    and durable queues work... There's some talk on the delayed exchange plugin about
    where the task is held before it posts to the queue. I'd have to read more into
    that. And on the persistence note, that lack of atomic task swap with rabbit irks
    me even more, as it's possible to lose jobs through that. That feels even worse
    once you expect jobs to persist across system restarts.

Security:
    I put a password in a source file for an email account for testing. Then I thought;
    oh I'll put it in a config file, and not distribute the config file. Then I remembered;
    it's now in source history even if I delete the head revision of the file. Obviously.
    Oops. Well I guess I'll just leave it there... one less thing to set up to run it...
