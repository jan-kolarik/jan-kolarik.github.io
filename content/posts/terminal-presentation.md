---
title: "Creating slides with integrated shell"
date: "2023-02-10"
summary: "How to setup a seamless interactive console within your presentation"
tags: ["revealjs", "ttyd", "podman"]
ShowCodeCopyButtons: true
ShowToc: true
ShowReadingTime: true
---

## Intro

If you are planning to make a presentation including some live command-line
examples, the following article could be useful for you. I'd like 
to share with you my setup which results in HTML presentation
containing embedded console windows which can run isolated from
your host system.

---

## Prerequisites

Following technologies are used to create the resulting presentation:

- [reveal.js](https://github.com/hakimel/reveal.js) - framework for creating HTML presentations
- [ttyd](https://github.com/tsl0922/ttyd) - tool to access Linux shell over HTTP
- [podman](https://github.com/containers/podman) - tool for managing containers

---

## Show the code first

### Prepare the example

Let's say we want to present and describe an example of our source code and
then show the audience how it is being run on the target system using the CLI.

Here we'll use this simple Python snippet:

```python
import random

# Generate the number
number = random.randint(1, 100)

# Print the number
print(f'Your lucky number is {number}.')
```

---

### Insert it inside the presentation

Suppose we have already configured and running some instance of the reveal.js 
presentation, we can insert a section with our code example there:

```html
<section>
    <h4>Python demo</h4>
    <pre>
        <code>
            import random

            # Generate the number
            number = random.randint(1, 100)

            # Print the number
            print(f'Your lucky number is {number}.')
        </code>
    </pre>
</section>
```

---

### Stylize the snippet

Now we can play a bit more with the layout. We could change the default `font-size`
and the `width` of the code block, so our snippet doesn't contain any scrollbars
and it is better centered within the presentation screen.

By default the reveal.js presentation has configured the highlight.js plugin
for syntax highlighting, so we can define the language of our snippet and apply
the colors by adding the `class="hljs language-python"`.

To emphasize only part of the code step by step, we can use the `data-line-numbers`
attribute where the vertical bar character denotes the transitions, f.e.
`"|1|3-4|6-7"` means starting with the whole code highlighted, followed by just
line number 1, then lines 3-4 and ending with lines 6-7.

The result could look like this:

```html
<section>
    <h4>Python demo</h4>
    <pre style="font-size: 18px; width: 60%;">
        <code class="hljs language-python" data-line-numbers="|1|3-4|6-7">
            import random

            # Generate the number
            number = random.randint(1, 100)

            # Print the number
            print(f'Your lucky number is {number}.')
        </code>
    </pre>
</section>
```

---

## Add an interactive console

### Setup the web terminal

Deploying the shell web server is very simple. When we have downloaded 
the ttyd binary, we just provide the port number where the daemon will be listening and
providing the HTTP layer above the console.

Following example will deploy ttyd web server on the port `1234` and for
every connected client it will create a new process with `bash`:

```bash
ttyd -p 1234 bash
```

---

### Integrate the console

Putting the console into the presentation is as simple as adding new `iframe`
pointing to our ttyd service at `http://localhost:1234/`:

```html
<iframe src="http://localhost:1234/"></iframe>
```

---

### Tuning the visual

Now to create a seamless transition between the code snippet and the
console window we need to add a bit more configuration.

We can setup a custom font size and also change the colors for ttyd
console like this:

```bash
ttyd -p 1234 -t fontSize=12 -t 'theme={"background": "white", "foreground": "black"}' bash
```

In reveal.js we will stack the console frame window on the top of the code snippet
while keeping it invisible until the snippet code slides are fully traversed. 
This could be done by including the both frames inside the parent `div` having the 
`r-stack` class and showing the console at the right moment by adding the `fragment` 
class to the console `iframe`.

In the end we can change the console frame size to match the code snippet.

One hack that could be handy **when we don't want to show the vertical scrollbar**
inside the console frame, but still keeping the scrolling functionality. In this case
we can wrap the console in the `div` which will match the size of the code example frame, 
but we stretch the width of the actual `iframe` a bit, so the scrollbar is hidden. This
also needs to setup `overflow: hidden;` in the wrapping `div`.

So the result could look like this:

```html
<section>
    <h4>Python demo</h4>
    <div class="r-stack">
        <pre id="code">
            <code class="hljs language-python" data-trim data-line-numbers="|1|3-4|6-7">
                import random

                # Generate the number
                number = random.randint(1, 100)

                # Print the number
                print(f'Your lucky number is {number}.')
            </code>
        </pre>
        <div id="cli-wrapper">
            <iframe id="cli" class="fragment fade-up" src="http://localhost:1234/"></iframe>
        </div>
    </div>
</section>
```

And the CSS now moved into it's own stylesheet:

```css
#code {
    font-size: 18px; 
    width: 60%;
}

#cli-wrapper {
    max-width: 60%; 
    width: 60%; 
    max-height: 100%; 
    height: 100%; 
    overflow: hidden;
}

#cli {
    max-width: 103%; 
    width: 103%; 
    max-height: 100%; 
    height: 100%;
}
```

**Note:** I am definitely not a CSS guy, so please don't blame me if you find
anything ridiculous about the mentioned code. But, it should work 😇

This is the final output when running the presentation in the web browser:

{{< video src="/posts/videos/revealjs-ttyd.webm" controls="yes" >}}

---

## Using containers

When doing more examples in the presentation it might be useful to always have
an isolated environment for each demo.

This can be done easily by using the podman containers. We can deploy a
container from the public image, do some customizations, prepare our demo
environment and then serialize the state of the container. Then we can setup
ttyd to run a clean container from this image every time client requests new 
console.

So if we use the Fedora Linux as an example, we can download the
latest Fedora container image and get inside that:

```bash
podman pull fedora
podman run -it fedora
```

It will redirect us inside the container terminal:

```bash
[root@fdee00b17d43 /]# 
```

Now prepare the environment needed for the demo, like installing dependencies,
copying the example scripts into the container etc.

When everything is ready, we can export the container from another shell:

```bash
podman container export fdee00b17d43 > container.tar
```

Then we can import it in the image registry on local or any other computer and 
tagging it with `example-container` name by doing:

```bash
podman image import container.tar example-container
```

Finally we will prepare the ttyd daemon to spawn a new container for us
on each attach:

```bash
ttyd -p 1234 podman run -it example-container /bin/bash
```

We can also change the container's hostname with `-h my-hostname`, so
the shell on the live demo will not show the ugly auto-generated id.

And of course we can prepare many containers running on different ports
with various font sizes, configuration, etc.

**Note:** each time client connects to the ttyd, new container is created.
This also means when the page having the embedded terminal is refreshed.
Therefore it may be desirable to cleanup all the related containers 
after the presentation is done:

```bash
podman container rm --filter ancestor=localhost/example-container
```

---