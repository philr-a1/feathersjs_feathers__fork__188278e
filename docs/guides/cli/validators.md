---
outline: deep
---

# Validators

For all currently supported schema types, AJV is used as the default validator. See the [validators API documentation](../../api/schema/validators.md) for more information.

## AJV validators

The `src/validators.ts` file sets up two Ajv instances for data and querys (for which string types will be coerced automatically). It also sets up a collection of additional formats using [ajv-formats](https://ajv.js.org/packages/ajv-formats.html). The validators in this file can be customized according to the [Ajv documentation](https://ajv.js.org/) and [its plugins](https://ajv.js.org/packages/). You can find the available Ajv options in the [Ajs class API docs](https://ajv.js.org/options.html).

```ts
import { Ajv, addFormats } from '@feathersjs/schema'
import type { FormatsPluginOptions } from '@feathersjs/schema'

const formats: FormatsPluginOptions = [
  'date-time',
  'time',
  'date',
  'email',
  'hostname',
  'ipv4',
  'ipv6',
  'uri',
  'uri-reference',
  'uuid',
  'uri-template',
  'json-pointer',
  'relative-json-pointer',
  'regex'
]

export const dataValidator = addFormats(new Ajv({}), formats)

export const queryValidator = addFormats(
  new Ajv({
    coerceTypes: true
  }),
  formats
)
```

## MongoDB ObjectIds

When choosing MongoDB, the validators file will also register the [`objectid` keyword](../../api/databases/mongodb.md#ajv-keyword) to convert strings to MongoDB Object ids.

## Custom Method Validation

When implementing [custom service methods](../../api/services.md#custom-methods), you can apply validation to the request and response data using the same validation mechanisms as standard service methods. This ensures that your custom methods maintain the same level of data integrity as the rest of your application.

Here's an example of how to implement validation for a custom method:

```ts
import { Ajv, schemaHooks } from '@feathersjs/schema'
import { Type, getValidator } from '@feathersjs/typebox'
import type { Static } from '@feathersjs/typebox'
import { dataValidator } from '../validators'

// Define the schema for the custom method request data
const resetPasswordSchema = Type.Object(
  {
    email: Type.String({ format: 'email' }),
    oldPassword: Type.String(),
    newPassword: Type.String({ minLength: 8 })
  },
  { $id: 'ResetPassword', additionalProperties: false }
)
type ResetPasswordData = Static<typeof resetPasswordSchema>

// Define the schema for the custom method response
const resetPasswordResponseSchema = Type.Object(
  {
    success: Type.Boolean(),
    message: Type.String()
  },
  { $id: 'ResetPasswordResponse', additionalProperties: false }
)
type ResetPasswordResponse = Static<typeof resetPasswordResponseSchema>

// Create validators for the request and response data
const resetPasswordValidator = getValidator(resetPasswordSchema, dataValidator)
const resetPasswordResponseValidator = getValidator(resetPasswordResponseSchema, dataValidator)

// Implement the custom method in a service class
class UserService {
  // Standard service methods...
  
  // Custom method for password reset
  async resetPassword(data: ResetPasswordData, params: Params) {
    // Implementation logic here
    return {
      success: true,
      message: 'Password successfully reset'
    }
  }
}

// Register the service with the custom method
app.use('users', new UserService(), {
  methods: ['find', 'get', 'create', 'update', 'patch', 'remove', 'resetPassword']
})

// Apply validation hooks to the custom method
app.service('users').hooks({
  around: {
    resetPassword: [
      schemaHooks.validateData(resetPasswordValidator),
      hooks.around(async (context, next) => {
        // Call the service method
        const result = await next()
        
        // Validate the response data
        schemaHooks.validateData(resetPasswordResponseValidator)(result)
        
        return result
      })
    ]
  }
})
```

In this example:

1. We define TypeBox schemas for both the request data (`resetPasswordSchema`) and the response data (`resetPasswordResponseSchema`).
2. We create validators for both schemas using `getValidator` with our application's `dataValidator`.
3. We implement the custom method `resetPassword` in our service class.
4. We register the service with the custom method name included in the `methods` array.
5. We apply validation to the custom method using `schemaHooks.validateData` in the `around` hooks.

This approach ensures that both the incoming data and outgoing responses for your custom methods are properly validated according to your defined schemas.
