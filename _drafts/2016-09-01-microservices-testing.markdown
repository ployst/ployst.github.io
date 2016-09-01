# TODO: Intro

# Organizing Microservices

When we first started ployst we had many repositories each with one
component. This had the advantage that any architectural cheating would be
more obvious (a PR exists to the 'wrong' repository), and github gave us the ability to assign different access rights that were person and repository dependent.

The disadvantages were that in order to make changes to many services as part
of one piece of work, many PRs had to be made. Additionally, every new service
created a significant overhead when being set up in CircleCI: All of the project
env vars needed to be set up, appropriate circle.yml etc.

As a result, we have taken the single repo approach to development.

This blog post will introduce a few of the things we're using to make this work
well in a CI-as-a-service environment.

# Testing Microservices

Following the idea of the [Test Pyramid](http://martinfowler.com/bliki/TestPyramid.html),
we have 3 layers of tests.

Within each component:
 - Unit tests (super fast. Don't touch the DB etc.)
 - Service tests (fast. these server to document the APIs)

Across components:
 - Integration tests (typically using the UI to do things. Requires multiple
                      components to be running)

We follow the rule that there should be many more tests in service than in
integration tests.

## Service tests

<!-- TODO: Images of services and service tests -->

Each service has some API that others are expected to communicate with. It's
important that each service tests that what it says it does it will do.

The problem is that, if service A talks to service B, and B's API changes, what
can happen is that B's tests reflect the change, all tests pass and it's merged.

But A is still using the older API.

Now, it's possible that this would be caught in the integration tests. And
hopefully it would be. But we really want to catch this kind of thing early:
The Test Pyramid shows that the lower level the test is, the cheaper it is.

That's where [Api-By-Example](https://github.com/apibyexample/abe-spec) has proven
invaluable.

Components define their api in json files, that show "given input x, i'll return
y". Then in tests components can use tooling like [abe-python](https://github.com/apibyexample/abe-python)
to confirm that they behave as documented.

But the real power of this is that - since we're in the same repository as all
other components - they too can use that file to confirm that they are sending
the correct things to that service, and handling the response data. Changes to
one component and its tests can now cause test failures at the service level
of another component. Quicker feedback means faster delivery!

# Continuous Integration

We're using CircleCI to build our services. Having a single repository in such
an environment raised some challenges.
 - circle only supports a single circle.yml file
 - build times were slow
 - feedback on PRs was binary (everything passed or failed)

A set of internal standards and build tools got round these problems.

## Build tools

We have a build tool, that checks to see which components have changed (compared to master) before performing the command.

    $ test $(git log origin/master..HEAD -- $COMPONENT_PATH | wc -l) -gt 0

When the builder's `test` command is run, all changed components have tests run on them,
and any dependent components (eg integration tests) will run against the changed
versions of components.

The build tool adds component-based labels to the pull request and reports back each step for greater granularity.

<TODO: image of PR feedback from circle>

## Standards

To build good tooling for our repo, we need to have a "build contract" that each
service adheres to. We call this the application-developer-interface (ADI), but
really it's the requirement that:
 - If the service wants to be built as part of a circle run it needs:
   - a make file in the root that can handle `make build` and `make test`
   - to appear in the central config of components (`builder.ini`)

That is all.

Here's an extract from our circle.yml file.

```yaml
dependencies:
  override:
    - pip install -e operations/tools/builder
    - ployst discover  # just to output what will be built
    - ployst build

test:
  override:
    - ployst test
```

And here's the sample builder.ini

```
[ployst-api]
downstream=integration-tests
path=ployst-api
release-process=docker

[ployst-ui]
downstream=integration-tests
path=ployst-ui
release-process=docker

[ployst-mailer]
path=ployst-mailer
release-process=docker

[integration-tests]
path=integration-tests

[builder]
path=operations/tools/builder

[deployer]
path=operations/tools/deployer
```

The path argument means that this can build libraries individually, without them
being at the top level.

We have a couple of libraries that are not released at all, but we want to run
tests.

The builder is responsible for building/testing everything in that file (that is running `make build` and `make test` on each of them).
If a component defines a release-process of docker, the builder is also responsible
for pushing the built-in-ci image to docker when `release` is called.

# Deployment

Whilst we're riding the alpha testing phase we're doing continuous production deployment to kubernetes.

To do this we use the build tool to perform the release to the docker registry of choice (in our case google's) and then kick off our deployer!


```yaml
deployment:
  production: # just a label; label names are completely up to you
    branch: master
    commands:
      - ployst release
      - cd operations/production && deployer deploy
      - ployst tag
```

`deployer deploy` can deploy dynamically rendered templates, and auto apply
static deployments. It is configured via `deployer.ini`.

```
$ cat deployer.ini
[deployer]
auto_release_dir=deployments/auto
templates_dir=deployments/templates

[ployst-mailer]
image=ployst-mailer
template_args=REPLICAS=1,MODE=live

...
```

## Dynamic Components (ie build dependent)

When running automatically on a master build, things like tag of the image
isn't known ahead of time. We use the CircleCI build number as the `DOCKER_TAG`
for all our images.
This means we need a template that can be rendered on the fly at deploy time.

Each component defined in deployer.ini will have its corresponding template
rendered using the env variables available and any additional template_args.

An example templated file is for the mailer:

```
$ cat operations/production/deployments/templates/ployst-mailer.yaml.erb
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: <%= ENV['APP'] %>-<%= ENV['MODE'] %>-deployment
spec:
  replicas: <%= ENV['REPLICAS'] %>
  strategy:
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: <%= ENV['APP'] %>
        mode: <%= ENV['MODE'] %>
    spec:
      containers:
      - name: ployst-mailer
        image: eu.gcr.io/ployst-proto/ployst-mailer:<%= ENV['DOCKER_TAG'] %>
        imagePullPolicy: IfNotPresent
        env:
        - name: RUN_MODE
          value: <%= ENV['MODE'] %>
```

The deployer renders this:
```
MODE=$MODE APP=$APP REPLICAS=$REPLICAS erb -r $ABS_TEMPLATES_PATH/templating $ABS_TEMPLATES_PATH/$DEPLOY_TEMPLATE > /tmp/${DEPLOY_TEMPLATE}_rendered
```

And then fires off the rendered yaml to Kubernetes:

```
cat /tmp/${DEPLOY_TEMPLATE}_rendered | kubectl apply -f -
```

## External, Static Components

Some components are static, or pre-determined. For example we have a OpenVPN container
that uses an image from docker hub. To upgrade this image we create a pull request that
includes an upgraded image tag and the deployer will release it on a master branch build
via `kubectl apply -f $AUTO_RELEASE_DIR`.


# Monitoring

We're using opbeat to monitor errors in production, log corresponding deployments,
and perform some basic performance monitoring.

<TODO: opbeat performance monitoring image>

The error handling is decent but the performance monitoring probably isn't in depth
enough in the long run so we'll need to augment it with other services.

# Future Improvements

We're really happy with the single repo, "change lots of things at once" approach.

However, it does mean that if someone has access to one component,
they have access to everything (even deployment definitions). I can foresee this
being less than ideal for some companies. A workaround may be to have n repos
that map one-to-one with your desired access control groups.

We're firing from the hip in the way we're rolling out changes to services/deployments
in the sense that we haven't automated the testing of a new set of kubernetes
templates.

We'll need to address that soon - to perhaps rollout entirely in a testing part of
circle to some test cluster, fire in smoke tests and if all seems to work then move
on.

How are you doing CI/deployments in kubernetes? Do get in touch and we can share
ideas.
