+++
title = "Logging in Node.js"
date = 2025-05-27T09:57:55+05:30
type = "post"
description = "A guide to logging in Node.js"
in_search_index = true
[taxonomies]
tags = ["nodejs", "observability"]
+++

## The Motivation
A few years ago, whenever I ran into a bug, my go-to solution was to add a console.log statement. I would make a change, save the code, and check the console to see what was happening. That was my entire debugging process.

Then I discovered the debugger in VSCode and Chrome DevTools. It completely changed the way I debugged. Instead of cluttering my code with logs, I could set breakpoints, step through the code, and watch variables in real time. It made everything so much easier.

But that only worked for local issues. When something breaks in the deployed environment, it's a different story. I can't just open the DevTools and step through the live server code. That's when I started thinking—how do I know what's going on in production?


## Wandering over it
At one point, I thought I had figured it out—I’ll just write the logs to a file. So I ran my server with `node server.js > logs.txt` and felt pretty smart. If something went wrong, I could just open the file and check what happened. But back then, I didn’t really understand how distributed systems worked. In production, there isn’t just one server running the code; there could be many. That meant each server would have its own logs.txt file, and I’d somehow need to collect logs from all of them. Yeah, not so simple anymore.

That’s when it hit me why people always say, “Get a mentor.” Because sure, you can try to figure things out on your own—but let’s be honest, you’re not Einstein. Sometimes you just need someone to point you in the right direction.

## Loggers in the ecosystem
There are a lot of loggers in the Node.js ecosystem like Winston, Bunyan, Pino, and others. But what do they actually do? What problems do they try to solve? I realized a good logger supports the following features:

* Ability to control the amount of logs it prints
* Pretty printing the logs in local environment
* Adding context to the logs
* Support for distributed logging across microservices
* Removing PII data from the logs
* Ease of use
* Working well together with other tracing libraries


What do i mean all the above features? Let's see.

Let’s say you have an e-commerce application! And let’s focus on the products service. How would it typically look?
```js
const logger = require('logger');

class ProductsController {
    constructor(private readonly productsService: ProductsService) {}

    async getProducts() {
        try {
            logger.info('Fetching products');
            const products = await this.productsService.getProducts();
            logger.info('Products fetched successfully', { products });
            return products;
        } catch (error) {
            logger.error('Error fetching products', { error });
            throw error;
        }
    }

    async createProduct(product: Product) {
        try {
            logger.info('Creating product', { product });
            const newProduct = await this.productsService.createProduct(product);
            logger.info('Product created successfully', { newProduct });
            return newProduct;
        } catch (error) {
            logger.error('Error creating product', { error });
            throw error;
        }
    }
}
```

#### Ability to control the amount of logs it prints
If we take the above code, my intention is to print the entry point, result of the operation, and any error that happens. Assume we are working with the scale of 1,000 requests per second. That means we are making 1,000 requests per second and logging 3 logs per request — that’s 3,000 logs per second.

Do we really need all these logs? Yes? No? It depends.

I want to see the complete picture in the development environment. But not in production. Why not in production?

* Because we are working at the scale of 1,000 requests per second, logs will surely add latency to the system.
* We are printing the whole product object, which might contain sensitive data.
* Logs are not free; they consume storage and bandwidth. We pay a lot for storing and searching in the logs.

So, we need to control the amount of logs we are printing. We can do that by using the logger.level property.

```js
logger.level = 'info';
```

It will only print the logs with the level info and above. In our case, it will print all the logs! 😂 Even when adding logs, we have to decide: where do we want to see it? Always? When something goes wrong? When debugging?

Let’s see how we can solve this.

##### Using different log levels for better control
```js
const logger = require('logger');

class ProductsController {
    constructor(private readonly productsService: ProductsService) {}

    async getProducts() {
        try {
            logger.debug('Fetching products');
            const products = await this.productsService.getProducts();
            logger.trace('Products fetched successfully', { products });
            return products;
        } catch (error) {
            logger.error('Error fetching products', { error });
            throw error;
        }
    }

    async createProduct(product: Product) {
        try {
            logger.debug('Creating product', { product });
            const newProduct = await this.productsService.createProduct(product);
            logger.trace('Product created successfully', { newProduct });
            return newProduct;
        } catch (error) {
            logger.error('Error creating product', { error });
            throw error;
        }
    }
}
```

Now, we are printing the logs with the level debug and above. It will print the entry point, but not the result of the operation since it’s a trace log. trace is lower than debug.

If we set:

```js
logger.level = 'debug';
```

It will print all logs with level debug and above, which means we get entry points and errors but skip detailed success traces. This reduces log volume while keeping useful info.

##### What do these logs look like?
Logs often include more than just a message — things like timestamps, trace IDs, and the data you want to capture. Here’s an example of how a few log entries might look in a log file (logs.txt):

```
{ timestamp: '2025-05-27T10:15:23.123Z', traceId: 'abc123', level: 'debug', message: 'Fetching products' }
{ timestamp: '2025-05-27T10:15:23.456Z', traceId: 'abc123', level: 'trace', message: 'Products fetched successfully', data: [{ id: 1, name: 'Coffee Mug' }, { id: 2, name: 'Notebook' }] }
{ timestamp: '2025-05-27T10:15:24.789Z', traceId: 'xyz789', level: 'error', message: 'Error fetching products', error: 'Database timeout' }
```

* timestamp tells you when the log was recorded
* traceId is a unique ID for the request, helping you trace logs across services
* level shows the log severity
* message is the main description
* data/error contain extra details relevant to the log entry


#### Pretty printing the logs in local environment





