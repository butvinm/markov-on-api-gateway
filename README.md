# Your API Gateway is Turing Complete!

You are probably familiar with the **Turing Machine** or **Lambda Calculus**—the foundations of functional programming and computer science in general.
What’s interesting is that around the same time, an alternative approach to computation was proposed in the USSR by **Andrey Markov Jr.**

He introduced a **string-rewriting system**, now called **Markov algorithms**.
Each program consists of a set of rules that define alphabetic sequences (called _patterns_) and corresponding sequences (called _substitutions_).
Program execution works by finding one of the patterns in the input string and replacing it with the corresponding substitution.
This process repeats until no more known patterns remain in the string.

Despite its simplicity, this system is **Turing-complete**, meaning it can compute any computable function.

Later, **V. Turchin** applied these ideas in his programming language **Refal**, extending them with more powerful and convenient patterns similar to **regular expressions (RegEx)**.

Today, every cloud provider offers an **API Gateway**. These gateways are typically configured using an **OpenAPI spec**, which allows defining actions for different request paths. 
They can match a request path against a pattern, extract parts of it into parameters, and substitute them into responses. One such response is an **HTTP Redirect**, which can send the request back to the same endpoint with a modified path.

So, an API Gateway can:

1. **Match** a pattern in the path,
2. **Replace** it with a new string,
3. **Repeat** this process until the path matches a specification.

Sound familiar? That’s right—an API Gateway’s specification can **run Markov Algorithms**, meaning it can theoretically execute any mathematical algorithm!

In this repository, I share my experiments implementing simple algorithms using **only** Yandex Cloud API Gateway’s path-matching and redirect capabilities.

> However, URL length limitations restrict the size of algorithms and program state.
> That said, we can use other techniques—such as transforming request and response bodies—to implement more sophisticated programs.

## Programs

You can find the programs I’ve implemented in the `./programs` directory.

### Rule 110

I managed to implement Rule 110, so we can definitely say that API Gateway is Turing complete

```shell
$ curl --max-redirs 250 -LsD - https://d5dlqokbr75c9u16bs4b.aqkd4clz.apigw.yandexcloud.net/rule110/_/_/_/_/_/_/_/_/_/_/_/_/_/_/o | grep -oP 'location: /start/\K.*'
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

## Usage

First, create a new API Gateway in Yandex Cloud. Follow the [official documentation](https://yandex.cloud/en/docs/api-gateway/quickstart/) for setup.

Next, load your program into the ~~computer~~ API Gateway:

```shell
yc serverless api-gateway update --id <gateway-id> --spec=./programs/sum.yml
```

To test it, send a cURL request:

- `-L` enables redirect following.
- `-sD -` provides a verbose log with all redirects.

```shell
curl -LsD - https://d5dlqokbr75c9u16bs4b.aqkd4clz.apigw.yandexcloud.net/sum/0/0/_/1/0
```

## Implementation details

Yandex Cloud API Gateway has specific [rules for path priority](https://yandex.cloud/en/docs/api-gateway/concepts/#algorithm).

These rules force us to use workarounds like redundant path parameters (e.g., n0, n1) to adjust path priorities.

Another caveat is wildcard parameters ({param+}), which capture empty paths—requiring us to add additional cases for the start and end of sequences.
