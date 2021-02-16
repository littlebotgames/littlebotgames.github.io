---
layout: post
title:  "Character Control - Actor / Controller"
date:   2021-02-16 20:00:00 +0000
categories: blog
---

## Your mind, body and soul

In **Little1** we use an **Actor / Controller** system to decouple the logic for controlling
an Actor (these are mostly characters but can be anything that needs controlling) from the rest of the Actor functionality. A nice way to think
about this is separating the concepts of the **brain** and the **body**.

This is pretty standard practice in games. [Unreal](https://docs.unrealengine.com/en-US/InteractiveExperiences/Framework/Controller/index.html)
even has this system built into the engine where they use the terms Controller and Pawn. 
The Controller can possess a Pawn and there is a 1 to 1 relationship between them.

Unity doesn't provide any of this functionality as standard but it's easy to get a simple implementation up and running.

### Actor

In **Little1** the Actor is very basic, all it has is a reference to the Controller that it can read it's inputs from. It's as simple as that.

The Actor MonoBehaviour lives on the Character GameObject.

~~~
public class Actor : MonoBehaviour
{
	[SerializeField]
	private Controller m_controller;
	public Controller Controller => m_controller;

	public void SetController(Controller controller)
	{
		m_controller = controller;
	}
}
~~~

### Controller

The Controller is just a store for input values. Systems which have access to the Actor can then get to the connected controller and 
the current input values.

If the Controller needs to be connected to different Actors then it can live on it's own GameObject, if it's specific to an Actor then it's
fine for it to live on the Actor GameObject.

~~~
public class Controller : MonoBehaviour
{
	public static class Button
	{
		public const int TorsoAction = 0;
		public const int LegsAction = 1;
		public const int Count = 2;
	}

	public static class Axis
	{
		public const int Move = 0;
		public const int Count = 1;
	}

	private bool[] m_pressed = new bool[Button.Count];
	private Vector2[] m_axes = new Vector2[Axis.Count];

	public void SetPressed(int id, bool value)
	{
		m_pressed[id] = value;
	}
	public bool GetPressed(int id)
	{
		return m_pressed[id];
	}

	public void SetAxis(int id, Vector2 value)
	{
		m_axes[id] = value;
	}
	public Vector2 GetAxis(int id)
	{
		return m_axes[id];
	}
}
~~~

For the player driven Controller these input values mirror input from the [Unity Input System](https://docs.unity3d.com/Packages/com.unity.inputsystem@1.0/manual/index.html). 
It's as simple as forwarding the input values from the Input System to the Controller.

~~~
public class PlayerBrain : MonoBehaviour
{
	[SerializeField]
	private Controller m_controller;
	private PlayerInput m_inputActions;

	private void Awake()
	{
		m_inputActions = new PlayerInput();
	}

	private void OnEnable()
	{
		m_inputActions.Enable();
	}

	private void OnDisable()
	{
		m_inputActions.Disable();
	}

	private void Update()
	{
		m_controller.SetPressed(Controller.Button.TorsoAction, m_inputActions.Player.TorsoAction);
		m_controller.SetPressed(Controller.Button.LegsAction, m_inputActions.Player.LegsAction);

		m_controller.SetAxis(Controller.Axis.Move, m_inputActions.Player.Boost.ReadValue<Vector2>());
	}
}
~~~

### Reacting to the input

Here is a very basic example of a script that would live on a Character GameObject alongside the Actor script. You can see it gets the Controller
from the Actor and applies motion to the Rigidbody based on the inputs from the Controller.

~~~
public class SimpleCharacterMotion : MonoBehaviour
{
	[SerializeField]
	private Actor m_actor;
	[SerializeField]
	private Rigidbody2D m_rigidbody;

	private const float c_moveSpeed = 10f;

	private void FixedUpdate()
	{
		var controller = m_actor.Controller;
		if(controller == null)
			return;

		var move = controller.GetAxis(Controller.Axis.Move);
		m_rigidbody.velocity = move * c_moveSpeed;
	}
}
~~~

### What's the point in all this then?

This might sound like we've just over complicated everything and made more work for ourselves. Now instead of just simply reading the input values
directly from the Unity Input System we have this extra step in the middle where we pipe the inputs through to the Controller.

But wait a minute, what's this? Thanks to this simple decoupling we now have the basic setup to do some amazing things. 

#### The magic

To make the player character be controlled by a simple AI all that is required is to write some code that sets the inputs in the Controller. 
The exact same logic used to move our player character already can still work.

~~~
public class BasicAIBrain : MonoBehaviour
{
	[SerializeField]
	private Controller m_controller;
	private float m_timer;
	private const float c_jumpSecs = 2f;

	private void Update()
	{
		// Walk forward and jump every x seconds
		var jumpPressed = false;
		m_timer -= Time.deltaTime;
		if(m_timer <= 0f)
		{
			jumpPressed = true;
			m_timer = c_jumpSecs;				
		}
		m_controller.SetPressed(Controller.Button.LegsAction, jumpPressed);
		m_controller.SetAxis(Controller.Axis.Move, new Vector2(1f, 0f));
	}
}
~~~

And it works the same the other way, after implementing AI for any enemies or NPCs in the game we are able to plug our player driven Controller in
and use the inputs from that instead of the inputs provided by the AI driven Controller. This gives us an incredibly powerful
debugging tool.

If we want any character to move in a cutscene we can now move them via a custom AI driven Controller which just provides the inputs
and they will move under the exact same constraints they would in the game. This means we don't have to pollute their normal AI behaviour
with cutscene specific logic, we just switch out the Controller instead.

And best of all if we decide tomorrow that we want to control a different character for a section of our game then it's easy to hook this up.
We no longer have a player character, we just have a character that is being controlled by the player. With a tiny bit of magical code 
we've just made the core gameplay mechanic of Super Mario Odyssey.

## Some bonus thoughts

In **Little1** instead of hardcoding the input actions into the Controller script we have a ScriptableObject where the actions can be defined.
This then auto generates code with the input action ids in so they can be used by any game code.

When making an AI driven Controller you will need to have knowledge of the Actor it is controlling so it can respond appropriately to what is happening
in game. This is why Unreal has the 1 to 1 relationship between it's Controller and Pawn. Player driven Controllers do not need to know about
their Actor as it is the player (human) who is responding to in game events.

This can feel counter-intuitive at times, especially in regards to axis inputs, because you are dealing with inputs instead of absolute values.
Instead of saying you want an Actor to move at a specific speed you are setting the input and relying on the Actor's motion calculations.
This gives a much better consistency to the motion but does mean you have to think a bit more when you want an Actor to do something like 
move to a specific spot. No more fudging the numbers.


## That's all folks

That's the 1st actual blog post done, if you have any questions / suggestions / comments then feel free to reach out on [Twitter](https://twitter.com/LittleBotGames)


Thanks for reading!

