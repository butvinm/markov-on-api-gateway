# Your API Gateway is Turing Complete!

## Historical Remark

You're probably familiar with the **Turing Machine** or **Lambda Calculus**—the foundations of functional programming and computation theory in general.
What’s interesting is that, around the same time, an alternative approach to computation was proposed in the USSR by **Andrey Markov Jr.**

He introduced a **string-rewriting system**, now called the [**Markov algorithm**](https://en.wikipedia.org/wiki/Markov_algorithm).
Each program consists of a set of rules that define _patterns_ and corresponding _substitutions_.
Program execution works by finding one of the patterns in the input string and replacing it with the corresponding substitution.
This process repeats until no more known patterns remain in the string.

Despite its simplicity, this system is [**Turing-complete**](https://en.wikipedia.org/wiki/Turing_completeness).

Later, **V. Turchin** applied these ideas in his programming language [**Refal**](https://en.wikipedia.org/wiki/Refal), extending them with more powerful and convenient pattern capabilities.

## The Idea

So, how is all of that related to API Gateways?

Today, every cloud provider offers an **API Gateway**, typically configured using a declarative specification like **OpenAPI**, which allows defining actions for different request paths.
They can match a request path against a pattern, extract parts of it into parameters, and substitute them into responses. One such response is an **HTTP Redirect**, which can send the request back to the same endpoint with a modified path.

So, an API Gateway can:

1. **Match** a pattern in the path,
2. **Replace** it to construct a new path with substituted parts,
3. **Repeat** this process by redirecting to the new path.

Sound familiar? That’s right—an API Gateway's specification can effectively **run Markov algorithms**, meaning it can theoretically execute **any** algorithm.
Yes, your API Gateway is actually [**Turing complete**](https://en.wikipedia.org/wiki/Turing_completeness)!

In this repository, you'll find experiments implementing simple algorithms using **only** Yandex Cloud API Gateway’s path-matching and redirect capabilities.

## How to Implement Algorithms on a Gateway

> Yandex Cloud is used here because it's more accessible for me, but you can probably reproduce this on any popular cloud provider.

Yandex Cloud API Gateway is configured using an extended version of the **OpenAPI** specification in YAML format.
We use the `dummy` extension to specify responses directly in the spec.

These features of the API Gateway and HTTP form the foundation of our programs:

1. API paths can contain wildcard patterns like `{param+}`, which match multiple segments of the path. A path can include several wildcard patterns.
2. Parameters captured with path patterns can be substituted into response content or headers.
3. The API can respond with a redirect (3xx) status code and specify a relative path in the `Location` header.

### How Yandex Cloud API Gateway Works

Before we write any programs, we need to understand how the API Gateway processes requests.

Wildcard patterns like `{head+}` will match as many segments as possible.
Given the path `/h0/h1/a/t0/t1` and pattern `/{head+}/a/{tail+}`, we capture `head=h0/h1` and `tail=t0/t1`.

However, there's a problem when a captured path is empty: after substitution, it may produce a malformed URL with double slashes (`//`).
Such paths won't match later, because double slashes aren't normalized.
This forces us to define extra handlers for paths where wildcard patterns are empty.

Another caveat is **path priority**.
When multiple paths match a request, [Yandex Cloud applies specific rules](https://yandex.cloud/en/docs/api-gateway/concepts/#algorithm) to determine which to use.
In short, to give wildcard paths higher priority, we need to artificially extend them with redundant patterns to increase both their segment count and character length.

So don’t be surprised when you see absurdly over-complicated paths like `/{tail0+}/{tail1+}` instead of a single `/{tail+}`.

### [A to B](./programs/ab.yml)

Let's start with a simple program that replaces all `/a` segments in the path with `/b`.

```yaml
openapi: 3.0.0
info:
  title: A to B
  version: 1.0.0
  description: Replace all /a/ path segments with /b/
paths:
  /a/{tail0+}/{tail1+}: # split {tail+} into two patterns to give this path higher priority
    get:
      parameters:
        - name: tail0
          in: path
        - name: tail1
          in: path
      x-yc-apigateway-integration:
        type: dummy
        http_code: 301
        http_headers:
          Location: /b/{tail0}/{tail1}
        content:
          "*": ""
  /{head+}/a/{tail+}:
    get:
      parameters:
        - name: head
          in: path
        - name: tail
          in: path
      x-yc-apigateway-integration:
        type: dummy
        http_code: 301
        http_headers:
          Location: /{head}/b/{tail}
        content:
          "*": ""
  /{result+}: # final path matches when there are no more `/a` segments
    get:
      parameters:
        - name: result
          in: path
      x-yc-apigateway-integration:
        type: dummy
        http_code: 200
        content:
          "*": "{result}"
```

The most important part of this algorithm is the second handler.

It matches paths with two wildcard segments surrounding an `/a` segment: `/{head+}/a/{tail+}`.
Then it redirects to a new path where `/a` is replaced with `/b`, and the other parts are kept unchanged: `/{head}/b/{tail}`.

The first handler processes `/a` at the beginning of the path to avoid producing empty segments in the redirect URL.

The last handler matches when there are no more `/a` segments. It returns the resulting path and terminates the program.

**Run:**

```shell
yc serverless api-gateway update --id <gateway-id> --spec=./programs/ab.yml
curl -L <gateway host>/a/n/a/n/a/s
b/n/b/n/b/s
```

> - `-L` flag is essential to enable redirect following.
> `-sD -` enable display of the full redirect chain and useful for debugging.

### [Binary sum](./programs/sum.yml)

This algorithm calculates the sum of two binary numbers with the same number of bits.
To invoke it, we use the following path format: `/sum/{a}/_/{b}` — for example:
`/sum/1/0/0/1/0/_/1/1/1/0/1`.

Essentially, we describe the truth table for two-bit addition with carry, and then compute the sum bit by bit, accumulating the result.

Let’s look at a single handler:

```yaml
/0/{a+}/0/_/{b+}/0/=/{c+}:
  get:
    # parameters
    x-yc-apigateway-integration:
      type: dummy
      http_code: 301
      http_headers:
        Location: /0/{a}/_/{b}/=/0/{c}
      content:
        "*": ""
```

The first segment represents the **carry**. Then we use two wildcard patterns, `{a+}` and `{b+}`, to extract the least significant bit and the rest of the arguments.
We define a separate handler for each combination of **carry**, `a₀`, and `b₀`.

The result bit is appended to the path segment captured by the `{c+}` wildcard.
The redirect location includes the new carry value, the `a` and `b` arguments reduced by one bit, and the extended result.

**Run:**

```shell
yc serverless api-gateway update --id <gateway-id> --spec=./programs/sum.yml
curl -L <gateway host>/sum/1/0/0/1/0/_/1/1/1/0/1
1/0/1/1/1/1
```

### [Rule 110](./programs/rule110.yml)

We can demonstrate Turing completeness by implementing [**Rule 110**](https://en.wikipedia.org/wiki/Rule_110), a one-dimensional cellular automaton that's proven to be Turing complete.

There is nothing interesting in the implementation, so lets focus on result. You can find full implementation of all algorithms in [programs/](./programs) directory.

**Run:**

```shell
yc serverless api-gateway update --id <gateway-id> --spec=./programs/rule110.yml
curl --max-redirs 250 -LsD - <gateway host>/rule110/_/_/_/_/_/_/_/_/_/_/_/_/_/_/o | grep -oP 'location: /start/\K.*'
_/_/_/_/_/_/_/_/_/_/_/_/_/_/o
_/_/_/_/_/_/_/_/_/_/_/_/_/o/o
_/_/_/_/_/_/_/_/_/_/_/_/o/o/o
_/_/_/_/_/_/_/_/_/_/_/o/o/_/o
_/_/_/_/_/_/_/_/_/_/o/o/o/o/o
_/_/_/_/_/_/_/_/_/o/o/_/_/_/o
_/_/_/_/_/_/_/_/o/o/o/_/_/o/o
_/_/_/_/_/_/_/o/o/_/o/_/o/o/o
_/_/_/_/_/_/o/o/o/o/o/o/o/_/o
_/_/_/_/_/o/o/_/_/_/_/_/o/o/o
_/_/_/_/o/o/o/_/_/_/_/o/o/_/o
_/_/_/o/o/_/o/_/_/_/o/o/o/o/o
_/_/o/o/o/o/o/_/_/o/o/_/_/_/o
_/o/o/_/_/_/o/_/o/o/o/_/_/o/o
o/o/o/_/_/o/o/o/o/_/o/_/o/o/o
```

## Limitations

URL length limits constrain the size of programs and their internal state, 
but no physical system can have infinite memory, so if the limitation of finite memory is ignored, our API Gateway is otherwise Turing-complete.
