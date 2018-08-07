# Iteration 1

For the first iteration, the application will be a very simple simulation, stripped to the bare bones as far as fancy stuff like graphics or algorithms are concerned: the focus here should just be on architectural structure.


## Business requirements

The application needs to simulate a ball bouncing on the floor:
- the ball is rigid and unconstrained
- the ball is subjected to constant gravity
- the floor is horizontal, fixed and rigid
- the floor is placed below the ball (with respect to gravity)
- the ball is perfectly elastic

This should capture the simplest possible simulation of a ball bouncing.

The ball needs to be rigid, so no deformation will need to be taken into account (elasticity is simulated theoretically).

The only force acting on the ball is gravity, meaning that the ball will just move vertically: this means that it's irrelevant if the simulation is in 1, 2 or 3 dimensions.

The floor is horizontal in the sense that it applies to the ball a reaction force that has same direction and opposite orientation than gravity.

Of course the floor must be below the ball, otherwise the ball would fall indefinitely.

The ball is perfectly elastic, so it will return at the same height as where it started falling, with respect to the floor.


### Nice to have

For future iterations, we might want to add the ability to store configuration values that the simulation depends on, like the height of the ball, or the value of gravity. These configuration values should be changeable (but just when the simulation is not running), and it should be possible to reset them to the original values.

Additionally, we want to keep a log of all major events happening in the system, which will be simulation start, simulation stop, and errors. This would be useful to perform an audit of the application once every while, to check that everything is working properly.


## Actors and use cases

Let's ignore the business model for now, and focus on the application. If we succeed in decoupling the domain layer from the application and infrastructure, we'll be able to implement the domain model without having to change application decisions too much anyway.

### Player

The first actor is the Player, who for now just Starts, Stops and Watches the Simulation. This is already an interesting situation, because usually we would think that the Start Simulation use case would automatically respond with some kind of output where the Player can Watch the Simulation. However, we could try to keep the Watch Simulation use case separated from the other two.

To do this, we can remember that input devices are generally different from output devices, at least in principle, and thus the application should keep the two separated, meaning that the output should be available in a different "channel" than the input. Then, if we are using a device that is capable of handling both input and output, like a GUI or console, its adapter will have to do what is required to correctly wire input and output to the two separate application use cases.

This is not much different than the CQRS approach used in business applications, where a command input doesn't return any meaningful output, and if you want to see how the state has changed after the command, you should issue the proper query.

In our case, we can think that the output is always available through some kind of "channel", and then the user can "connect" to that channel when he wants to check the output. Thus, the Start Simulation use case will just let the Simulation begin, without returning any kind of output. After having Started the Simulation, the user should perform the Watch Simulation use case to connect to the output stream containing the Simulation Data that is being generated. Of course the Stop Simulation use case will end the Simulation, and stop the data flux as well.

```gherkin
Feature: Simulation
    As a Player
    I want to Run the simulation
    In order to Watch it
    
    Scenario: Watching Not Running Simulation
        Given the Simulation is Not Running
        When I Watch the simulation
        Then I get an Error Message about Watching Not Running Simulation
        
    Scenario: Start the Simulation
        Given the Simulation is Not Running
        When I Start the Simulation
        And I Watch the Simulation
        Then I don't get any Error Messages
        
    Scenario: Starting Running Simulation
        Given the Simulation is Running
        When I Start the Simulation
        Then I get an Error Message about Starting Running Simulation
        
    Scenario: Watch the Simulation
        Given the Simulation is Running
        When I Watch the Simulation
        Then I can fetch the Simulation Data
        
    Scenario: Stop the Simulation
        Given the Simulation is Running
        When I Stop the Simulation
        And I Watch the Simulation
        Then I get an Error Message about Watching Not Running Simulation
        
    Scenario: Stopping Not Running Simulation
        Given the Simulation is Not Running
        When I Stop the Simulation
        Then I get an Error Message about Stopping Not Running Simulation
```

Here we've decided that the application would throw an error when trying to Watch a Not Running Simulation. Alternatively, we could add another use case like Check Simulation State, but this is not part of the business requirements: it would only be a way to simplify the design. 


### Auditor

The Auditor is the one interested in checking audit logs. Concerning use cases, there isn't really any direct use case for the Auditor, since he would likely not use the application to check its logs, however we can ensure that logs are written at the right times, by pretending the Auditor is using the application to verify that logs are written. In actuality, the application will be concerned with sending Messages, rather than writing logs, so what we want to check is really that the proper Messages are Sent.

```gherkin
Feature: Audit Messages
    As an Auditor
    I want to check Messages for all major events
    In order to verify that the system is working properly
    
    Scenario: Message Sent Watching Not Running Simulation
        Given the Simulation is Not Running
        When I Watch the Simulation
        Then a Message is Sent about Watching Not Running Simulation
        
    Scenario: Message Sent Starting Simulation
        Given the Simulation is Not Running
        When I Start the Simulation
        Then a Message is Sent about Starting the Simulation
        
    Scenario: Message Sent Starting Running Simulation
        Given the Simulation is Running
        When I Start the Simulation
        Then a Message is Sent about Starting Running Simulation
        
    Scenario: Message Sent Stopping Simulation
        Given the Simulation is Running
        When I Stop the Simulation
        Then a Message is Sent about Stopping the Simulation
        
    Scenario: Message Sent Stopping Not Running Simulation
        Given the Simulation is Not Running
        When I Stop the Simulation
        Then a Message is Sent about Stopping Not Running Simulation
```


### Administrator

Another role that might be needed is that of the Administrator, that will be able to Manage the Configuration of the Simulation. Here the use cases will generically be Read Configuration, Update Configuration and Reset Configuration:

```gherkin
Feature: Configuration
    As an Administrator
    I want to Manage the Configuration
    In order to tweak the way the Simulation Runs
    
    Scenario Outline: Read Configuration
        When I Read the value of Configuration <conf>
        Then the value <value> is returned
        
        Examples:
            | conf        | value |
            | ball-height | 10    |
            | gravity     | 9.81  |
    
    Scenario Outline: Update Configuration
        Given the value of Configuration <conf> is <value>
        When I Update the Configuration <conf> with <new>
        And I Read the value of Configuration <conf>
        Then the value <new> is returned
        
        Examples:
            | conf        | value | new |
            | ball-height | 10    | 20  |
            | gravity     | 9.81  | 5.3 |
    
    Scenario Outline: Reset Configuration
        Given the value of Configuration <conf> is <value>
        And I Update the Configuration <conf> with <new>
        When I Reset the Configuration
        And I Read the value of Configuration <conf>
        Then the value <value> is returned
        
        Examples:
            | conf        | value | new |
            | ball-height | 10    | 20  |
            | gravity     | 9.81  | 5.3 |
```
![Actors diagram](./usecase.svg)


## Glossary

- Simulation Data: set of data representing the evolution over time of the domain-specific physics model. 
- Simulation: production of Simulation Data.
- Start Simulation: to let the Simulation start producing Simulation Data, that can thus be fetched after it.
- Running Simulation: a Simulation that is producing Simulation Data, after having been Started.
- Stop Simulation: to make the Simulation stop producing Simulation Data, that cannot thus be fetched anymore after it.
- Not Running Simulation: a Simulation that is not producing Simulation Data, either because it wasn't Started, or because it was Stopped.
- Watch Simulation: to fetch the Simulation Data in order to present it in some way.
- Run Simulation: perform the lifecycle of a Simulation, meaning Starting it and then Stopping it at some point.
- Message: a means for the application to deliver a text message. This can be caught by devices attached to the application, which can in turn do something with it, like displaying it, or logging it.
- Error Message: a Message reporting some kind of error that happened within the application.
- Send Message: delivering a Message to the outside of the application.
- Configuration: set of values used to determine how the Simulation Runs.
- Manage Configuration: to act upon the Configuration in order to change it.
- Read Configuration: to retrieve a specific Configuration value that is currently set.
- Update Configuration: to change a specific Configuration value.
- Reset Configuration: to change Configuration values to the default ones.
- Player: an actor who can Start and Stop the Simulation, and Watch it.
- Auditor: an actor who checks Messages to verify that the application is functioning properly.
- Administrator: an actor who can Manage Configuration. 
