![Banner Light](./.assets/banner-vitest-mock-commonjs-light.png#gh-light-mode-only)
![banner Dark](./.assets/banner-vitest-mock-commonjs-dark.png#gh-dark-mode-only)

# vitest-mock-commonjs

## Overview

I started writing more vitest scenarios for Auth0 Action Scripts and Auth0 does not support ESM in Action Scripts, only CommonJS.
So I became frustrated trying to work around the limitation in vitest that it won't shim the Node.js "require" statement so that
I can mock a CommonJS module.
You may be wondering about the name, but the goal is not to mock "require", it is to shim require so that we can mock CommonJS modules!

I understand the problems facing the maintainers of vitest:
they are focused on ESM, and if you are using a package framework with its own module loader it may not be easy to hook into and shim
to override the loading of CommonJS modules.
And many folks may not consider it important to use both ESM and CommonJS at the same time.
But in the case of writing tests for Auth0 Action Scripts that is exactly what I want to do!
The scripts must be written in JavaScript with CommonJS, but I want the test suites to be written using ESM in either JavaScript
or TypeScript!

The CommonJS module loader for Node.js is well defined and that is what I focused on first.
Well, maybe not well defined in the documentation, but how it works is common knowledge!
So, to start I created a Node.js package that can be used to shim a "mockForNodeRequire" method onto the vi object in vitest
to mock "requires" in the code under test.
This module is hidden behind a controller in index.js, which allows multiple shims for other module loaders
to be injected into vitest using a common framework.

## Configuration and Use

### Supported module loaders

| Framework Name | vi.{Method} or Function Name |
|---|---|
| Node.js | mockForNodeRequire |

### Installation

With npm or yarn import this module into the test module where you want to mock CommonJS modules.
Since this is intended for unit testing, import the module as a development dependency.
The module overrrides the module loader and allows the mocking of multiple CommonJS modules.

```
$ npm install --save-dev vitest-mock-commonjs
```

This module depends on the "latest" version of vitest; the test suite also needs to do that to make
sure it is using the same version or the injected methods will not be found.
This happens because npm and yarn support managing multiple module versions adjacent to each other
in node_modules.

The package.json over the test-suite should specify "latest" as the version for vitest:

```
{
    "devDependencies": {
        "vitest-mock-commonjs": "latest"
        "vitest": "latest"
    }
}
```

### Inclusion in the test suite

Import vitest and this module in the test suite.
The order of import is not important, vitest-mock-commonjs will import vi itself to inject the methods.
What other exports from vitest you import are up to you, this is just an example:

```
import { beforeAll, beforeEach, vi } from 'vitest'
import 'vitest-mock-commonjs'
```

vitest-mock-commonjs also exports the mock creation functions for direct use, simply import the named functions:

```
import { mockForNodeRequire } from 'vitest-mock-commonjs'
```

### Mocking a CommonJS module

Mock CommonJS modules by calling the mock creation function.
This example shows both forms.
{} is used as a placeholder here for the actual test double:

```
vi.mockForNodeRequire('auth0', {}) // or directly with mockForNodeRequire('auth0', {})
```

TypeScript declarations are included in the index.d.ts file, so this should be directly useable
with TypeScript.

### The test double (a side-trip on "how to mock a CommonJS module")

The test double depends on the what the CommonJS module does.
The auth0 module is a good multi-level example.
The module itself does not have a default export.
The ManagementClient property is a class that produces an object with a users property which
is all we are interested in at the moment.
The test double must be in global space so that the mocked methods can be referenced in the tests.
It is probably best to "hoist" them too, in case there is anything else in a common declaration
used for mocking or spying:

```
const mocks = vi.hoisted(() => {

    const managementClient = {

        users: {

            delete: vi.fn(async (requestParameters) => new Promise((resolve) => resolve()))
        }
    }

    class ManagementClient {

        constructor() {

            this.users =  managementClient.users
        }
    }

    const mocks = {

        auth0Mock: {
            
            ManagementClient: ManagementClient,
            managementClient: managementClient
        }
    }

    return mocks
})
```

If you follow this carefully, the class definition and the static object the class is always evaluated as (for mocking) are
attached to a property hoisted before using vitest and available in the global "mocks" variable.
That way they can both be referenced for "expect" assertions in the tests!

We only need to mock the CommonJS module(s) once.
There is no requirement to hoist this call, so that could be done in the hoisted code or more likely it could be placed in
a definition of *beforeAll*:

```
describe('Action tests', async () => {

    beforeAll(async () => {

        vi.mockForNodeRequire('auth0', mocks.auth0Mock)
    })
```

All that is left is checking to see if the function was actually called in the code under test:

```
    it('Rejects authentication when the only deny entry is the denied user', async () => {

        mocks.eventMock.secrets.deny = 'calicojack@pyrates.live'
        mocks.eventMock.user.email = 'calicojack@pyrates.live'

        await onExecutePostLogin(mocks.eventMock, mocks.apiMock)

        expect(mocks.auth0Mock.managementClient.users.delete).toHaveBeenCalled()
    })
```

There are test double definitions missing in the example, hinted at by the individual test case above.
The whole working, stand-alone example that uses vitest-mock-commonjs and from which
these examples were pulled can be seen at https://github.com/jmussman/auth0-block-idp-signup. 

### The code-under-test (CUT)

The mocked CommonJS module will be loaded in the code under test because the module loader has
been overridden to do so.
This is what the CUT from the example looks like, and there is nothing in there to knowingly support
the test (a fundamental principles of testing):

```
const ManagementClient = require('auth0').ManagementClient;

const managementClient = new ManagementClient({

    domain: event.secrets.domain,
    clientId: event.secrets.clientId,
    clientSecret: event.secrets.clientSecret,
});

...

await managementClient.users.delete({ id: event.user.user_id });
```

## Implementation (how it works)

vitest fits with ESM very well, while jest fits with CommonJS.
Because the focus is ESM, vitest-mock-commonjs is only delivered as an ESM module.
Of course an index.d.ts file is provided for TypeScript compatibility.

### 

This model defines a single function that is used to mock CommonJS modules but
will also provide the shim for those test doubles to work:

```
async function mockForNodeRequire(module, testDouble) {
}

vi.mockForNodeRequire = mockForNodeRequire

export default mockForNodeRequire
```

### Multiple Module Mocking

This model extends some examples that are found around the Internet, but adds mocking multiple
CommonJS modules at the same time.
An object named *testDoubles* is attached to the function object to remember the test double
for each module, using the module name as an object property name.
Strict mode blocks a standalone function from having a *this* pointer that points to itself;
to get arround this the function is named using the classic JavaScript declaration and the name references
the function within itself:

```
async function  mockForNodeRequire(module, testDouble) {

    if (!Object.getOwnPropertyNames(mockRequire).testDoubles) {

        mockForNodeRequire.testDoubles = {}
```

### Overridding Module._load

Overriding the Node.js module loader is fairly simple with ESM.
In this model the override is handled in the function that will be used for mocking, the first time the
function is called.
To override Node.js "module" is imported and the _load function overridden.
After the override, or if it was already overridden, the test double is added to the testDoubles object.
When require is called in the CUT, the new _load method looks at the module requested and either serves
the double or uses the original _load method to serve the actual module if no double exists:

```
async function  mockForNodeRequire(module, testDouble) {

    if (!mockForNodeRequire.testDoubles) {

        mockForNodeRequire.testDoubles = {}

        const { Module } = await import('module')

        Module._load_original = Module._load

        Module._load = (uri, parent) => {

            const result = mocNodeRequire.testDoubles[uri] ?? Module._load_original(uri, parent)

            return result
        }
    }

    if (module && testDouble) {

        mockForNodeRequire.testDoubles[module] = testDouble
    }
}

mockForNodeRequire()

export default mockForNodeRequire
```

There is a conditional check provided to avoid trying to record a double when no module is specified.
The shim does need to be hoisted, but if the function is called without a module argument during
initialization the the shim is hoisted anyways because module initialization is hoisted.
You may wonder why do that at all, why not just define the shim outside of the function?
We keep it as a lambda inside so that it has closure access to the testDouble property!

The last step is the index.js module.
This loads all of the defined mock creation modules for different module loaders and attaches them
too the vi object:

```
import { vi } from 'vitest'
import mockForNodeRequire from '../src/mockForNodeRequire'

vi.mockForNodeRequire = mockForNodeRequire

export { mockForNodeRequire }
```

## License

The code is licensed under the MIT license. You may use and modify all or part of it as you choose, as long as attribution to the source is provided per the license. See the details in the [license file](./LICENSE.md) or at the [Open Source Initiative](https://opensource.org/licenses/MIT).


<hr>
Copyright © 2024 Joel A Mussman. All rights reserved.
