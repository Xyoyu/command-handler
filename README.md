# JADL Command Handler

## This new version of the command handler is WIP and not stable, use with caution.

# Prerequisites

If you're going to use the decorators built into the command handler, make sure to set `"experimentalDecorators": true` in your tsconfig.json

## Creating a Command Handler

Create a command handler with the CommandHandler class

```ts
import { CommandHandler } from '@jadl/cmd'

import { WaveCommand } from './commands/WaveCommand' // example command

const commands = new CommandHandler(worker, [
  // your command classes
  WaveCommand
])
```

## Creating a command

Commands are classes that will be filled with decorator to dictate their functionality

Our main decorator for a command is `@Command(name: string, description: string)`

Once you've declared your class, you create a method to run the command through, and mark it with a `@Run()` decorator

You can return a traditional MessageTypes and it will respond to the command with that. Here's an example;

```ts
import { Command } from '@jadl/cmd'

@Command('wave', 'Wave at someone!') // sets the command name to /wave
export class WaveCommand {
  @Run() // marks this method as the main running command
  wave () {
    return 'Hello!'
  }
}
```

Simplest command in the book. Lets get a bit more advanced.

### Accessing data with parameter decorators

Using parameter decorators is how we access all of our important data, to make your command a real command

Many of them exist and allow for adding and using interaction options, or just accessing your normal data objects

e.g let's add a user via the `Options.User` decorator. This will create a Discord interaction user option

```ts
import { Command, Options } from '@jadl/cmd'
import { Embed } from '@jadl/embed' // optional, but used for embeds!

@Command('wave', 'Wave at someone!')
export class WaveCommand {
  @Run()
  wave (
    @Options.User('user', 'User to wave at') userId: Snowflake // creates an option accepting type user
  ) {
    // you can now use this parameter as it's actual value! making it super easy to do what you need to do
    return new Embed()
      .description(`Hey <@${userId}>! Someone waved at you`)
  }
}

// note, discord-api-types is EXTREMELY useful and will let you add types for all of these
```

Some helpful decorators:

- `@GetWorker`, gets the main worker
- `@Interaction` gets the interaction object
- `@Guild` gets the guild
- `@Author` gets the running user
- `@Member` gets the running member

(many more to come) // WIP

### Middleware / interceptor decorators

With decorators it's super easy to create and use decorator for specific commands. There's a few built in. For example `@UserPerms()`

```ts
@Command('ban', 'Ban a user')
class BanCommand {
  @Run()
  @UserPerms('administrator') // this will require the user has the administrator permission
  run () {
    // run ban
  }
}
```

## Creating your own

You can also make your own decorators with the `Decorators` object

There are 3 types of decorators

### Base Decorators

Base decorators go above the command class similar to the `@Command()` decorator. These are mostly used for internal things, but it's available

You can create this with the `Decorators.createBaseDecorator()` method

### Command Decorators

Command decorators are the decorators that go over the method being ran, and are generally used as middleware / interceptors, or just defining extra metadata about the command

You can create these with the `Decorators.createCommandDecorator()` method

e.g let's create a specific user only decorator

**src/decorators/UserLocked.ts**
```ts
import { Decorators } from '@jadl/cmd' // global utility for everything decorators

export const UserLocked = Decorators.createCommandDecorator<[
  // options
  userId: string
]>(([userId], cmd) => {
  // Use the canRun array of functions
  cmd.canRun.push((interaction, worker) => { // interaction being the raw object
    // return a boolean of whether or not the author is the user locked
    return interaction.user.id === userId
  })
  // you can also use the onRun array of functions to disregard returning and errors
})
```
You can now use this in your command like so

```ts
import { UserLocked } from '../decorators/UserLocked'

@Command('wave', 'Wave at someone!')
export class WaveCommand {
  @Run()
  @UserLocked('277183033344524288') // this will now apply the above canRun method
  run () {
    return 'Wave!' // only ran if the user matches the user locked to this command
  }
}
```

### Parameter decorators

Parameter decorators are used to pass data to the running method when a command has been ran

The essential way this works is that you're giving the command handler a function that will be ran and positioned to your method based on the parameter

You can create these with the `Decorators.createParameterDecorator()` method

e.g let's make a database decorator

**src/decorators/Db.ts**
```ts
import { Decorators } from '@jadl/cmd'

// you can add options the same was as was done above, however we don't need that here
export const Db = Decorators.createParameterDecorator((options) => {
  return async (interaction, worker) => { // this method is ran EVERYTIME a command is ran, and it's return value is what shows up on the parameter for your method
    return await worker.db.guildSettings.get(interaction.guild_id) // returns the guild's database
  }
})
```
Now we can use the `@Db()` decorator shorthand in our method

```ts
import { Db } from '../decorators/Db'

@Command('wave', 'Wave at someone!')
export class WaveCommand {
  @Run()
  wave (
    @Db() db: GuildSettings, // makes our db parameter
    @Author() author: APIUser // and you can add as many of these params as you'd like!
  ) {
    // and now db will be whatever was returned in the Db decorator!
    if (!db.users.includes(author.id)) return "You can't do that!"

    return 'Wave!'
  }
}
```