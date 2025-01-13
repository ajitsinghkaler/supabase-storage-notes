# Monitoring

logflare

we get logflare env varables . After that we crete a log stream using logfalre and return it.

logger.ts

In logger.ts we create a pino logger with build transport our logfalre.ts for error we normalize errror using error.ts Then we whileist all headers by making it as sting and replacing - with _. For request also we get various things taht we need for request we redact query with are user related like token and X-Amz-Security-Token etc

metrics.ts

We set creates a new Prometheus registry where all metrics will be registered. we use it in agent.ts and various other places

otel-instrumentation.ts

Only two methods are important rest set config. patchMehod We get all the methods taht we need to to path with open telementry and we run aloop over them .In each methoid we add a method on protype of object. In this we start a new span set various config and aslo set the original without monkey pataching.   mankey patch the original method to set  various tracer things. and call original method. Aftre this when we unpatch we set the methiod top original methiod.

otel-processor.ts

The onStart method is called whenever a new span begins in your application. It receives a ReadableSpan object which contains the tracing information. The method handles two main scenarios:

First, it handles root spans (spans without parents) by checking `!span.parentSpanId`. When encountering a root span, it either creates a new trace entry in the traces Map if one doesn't exist, or ignores it if the trace is already being tracked. A new trace entry consists of the trace ID, an array of spans (initially containing just the root span), and the root span's ID.

Second, for child spans (spans with parents), it locates the existing trace using the trace ID and then calls addChildSpan to insert the span into the correct position in the span tree hierarchy.

The addChildSpan method uses recursive traversal to build the span tree. It searches through the spans array looking for the parent span (matching by span ID). When found, it adds the child span to that parent's children array. If the parent isn't found at the current level, it recursively searches through each span's children until the parent is located.

otel.ts

This is just setup for open telementary. We set it up on varios methods on various classes and http. Wes top some default instrumentations to reduce noise.