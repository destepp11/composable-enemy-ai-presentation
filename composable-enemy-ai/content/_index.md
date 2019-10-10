+++
title = "Applying Composition to Enemy State Machines"
description = "A pluggable architecture for enemy state machines."
outputs = ["Reveal"]
[reveal_hugo]
custom_theme = "reveal-hugo/themes/robot-lung.css"
margin = 0.2
highlight_theme = "color-brewer"
transition = "slide"
transition_speed = "fast"
[reveal_hugo.templates.hotpink]
class = "hotpink"
background = "#FF4081"
+++

![composition](images/construx.png)

# Composition and Enemy State Machines

----

Dennis Stepp
<br>
![profile-pic](images/about/portrait-mrmcgigglets.png)

Product Owner
<br>
![profile-pic](images/about/lirio.png)

Developer
<br>
![profile-pic](images/about/unibear-bear.png)

----

## What's a state Machine?

A device which can be in one of a set number of stable conditions depending on its previous condition and on the present values of its inputs.

----

![not-helpful](images/not-helpful.png)

----

![game-state](images/game-state.png)

----

![player-state](images/player-state.png)

----

![enemy-state](images/enemy-state.png)

----

```
void Update ()
{
  // Move the enemy
  Vector3 destination = controller.wayPointList[controller.nextWayPoint].position;
  while (Vector3.Distance(controller.transform.position, destination) > 0.1f)
  {
    // if the player is detected shoot at the player
    if (IsPlayerDetected)
    {
      Debug.unityLogger.Log("Shoot the player.");
      GameObject shotObject = (GameObject)Instantiate(shotPrefab, transform.position + transform.right * 0.5f, transform.rotation);
      shotObject.transform.Rotate(0f, 0f, Random.Range(-spreadOfShots, spreadOfShots));
    }
    Debug.unityLogger.Log("Traveling towards the waypoint");
    controller.transform.position = Vector3.MoveTowards(controller.transform.position, destination,
    controller.sentinelStats.moveSpeed * Time.deltaTime);
  }
  else
  {
    Debug.unityLogger.Log("Incrementing the waypoint");
    controller.nextWayPoint = (controller.nextWayPoint + 1) % controller.wayPointList.Count;
  }

  Vector3 movement = new Vector3 (moveHorizontal, 0.0f, moveVertical);

  rb.AddForce (movement * speed);
}
```

----

![actions-and-states](images/here-actions-and-states.png)

----

![decision](images/here-decision.png)

----

![decision](images/here-data.png)

----

![actions-and-states-and-decisions](images/here-actions-and-states-and-decisions.png)

----

![actions-and-states-and-decisions-and-data](images/here-actions-and-states-and-decisions-and-data.png)

----

## Let's Refactor to use `IEnumertor`

----

Our `Patrol` action is pretty basic. Let's make it more interesting. 

----

```
        private IEnumerator Patrol()
        {
            _spotlight.SetTargetColor(Colors.NomorianBlue);
            while (Vector3.Distance(transform.position, WaypointPositions[_currentWaypoint]) > 0.1f)
            {
                if (Vector3.Angle(transform.right, WaypointPositions[_currentWaypoint] - transform.localPosition) < 1f)
                {
                    _rb.velocity = transform.right * Speed;
                }
                else
                {
                    _rb.velocity = _rb.velocity * Time.deltaTime;
                }

                RotateTowardsTarget(WaypointPositions[_currentWaypoint]);

                if (IsTargetDetected())
                {
                    JumpToSeek();
                }
                yield return null;
            }

            _currentWaypoint++;
            if (_currentWaypoint == WaypointPositions.Length)
            {
                _currentWaypoint = 0;
            }
            StartCoroutine("Scan");
        }
```

----

![patrol-breakdown](images/patrol-breakdown.png)

----

```
private IEnumerator Fire()
        {
            WeaponSprite.SetActive(true);
            _spotlight.SetTargetColor(Colors.NomorianOrange);
            _timeOfLastShot = Time.time;
            while (IsPlayerInRange() && !IsPlayerObstructed())
            {
                RotateTowardsTarget(Game.instance.playerContainer.playerPhysics.position);
                if (Time.time > _timeOfLastShot + TimeBetweenShots)
                {
                   GameObject shotObject = (GameObject)Instantiate(ShotPrefab, transform.position + transform.right * 0.5f, transform.rotation);
                   shotObject.transform.Rotate(0f, 0f, Random.Range(-SpreadOfShots, SpreadOfShots));
                    _timeOfLastShot = Time.time;
                }
                yield return null;
            }
            WeaponSprite.SetActive(false);
            StartCoroutine("Scan");
        }
```

----

![shoot-breakdown](images/shoot-breakdown.png)

----

If these functions look like this, then what does the entire class look like?

----

![class-breakdown](images/class-breakdown1.png)

----

![class-breakdown](images/class-breakdown2.png)

----

![class-breakdown](images/class-breakdown3.png)

----

![class-breakdown](images/class-breakdown4.png)

----

![class-breakdown](images/class-breakdown5.png)

----
## Hierarchical Domain

Let's break down what a behavior is into five aspects.

**Actions**, **States**, **Decisions**, **Transitions**, **Data**

----

**Actions** are the process of doing something.

----

**States** are a particular condition that something is in at a specific time.

----

**Decisions** are the act of choosing to alter one's state by taking some action to reach some outcome.

----

**Transitions** are the act of carrying out a decision to take a new action.

----

**Data** are elements such as integers, strings, sprites, game objects, and other objects.

----

![domain](images/domain.png)

----

Unity's `ScriptableObject` turns code into assets we can use as building blocks for our state controller.

---- 

## Data

```
[CreateAssetMenu (menuName = "ComposableAI/BaseEnemyData")]
  public class BaseEnemyData : ScriptableObject
  {
    public float movementSpeed;
    public float lookRange = 40f;
    public float lookSphereCastRadius = 2f;
    public float chaseRange = 20f;
    public float attackRange = 5f;
    public float attackRate = 4f;
    public float attackForce = 15f;
    public float searchDuration = 4f;
    public float searchingTurnSpeed = 120f;
    public float substantialTargetVolumeThreshold = 100f;
  }
```

----

## Action

```
public abstract class Action : ScriptableObject
{
  public abstract void Act(StateController controller);
}
```

----

## Patrol Action

```
[CreateAssetMenu (menuName = "PluggableAI/Actions/Patrol")]
public class PatrolAction : Action
{
  public override void Act(StateController controller)
  {
    Patrol(controller);
  }

  private void Patrol(StateController controller)
  {
    Debug.unityLogger.Log("Entered the Patrol Action");
    Vector3 destination = controller.wayPointList[controller.nextWayPoint].position;

    if (Vector3.Distance(controller.transform.position, destination) > 0.1f)
    {
      Debug.unityLogger.Log("Traveling towards the waypoint");
      controller.transform.position = Vector3.MoveTowards(controller.transform.position, destination,
      controller.enemyStats.moveSpeed * Time.deltaTime);
    }
    else
    {
      Debug.unityLogger.Log("Incrementing the waypoint");
      controller.nextWayPoint = (controller.nextWayPoint + 1) % controller.wayPointList.Count;
    }
  }
}
```

----

## Shoot Action

```
[CreateAssetMenu(menuName = "PluggableAI/Actions/Shoot")]
  public class ShootAction : Action
  {
    public override void Act(StateController controller)
    {
      Shoot(controller);
    }

    private void Shoot(StateController controller)
    {
      controller.enemyStats.timeOfLastShot = Time.time;
      Debug.unityLogger.Log("Shoot the player.");
      GameObject shotObject = (GameObject)Instantiate(controller.shotPrefab, controller.transform.position + controller.transform.right * 0.5f, controller.transform.rotation);
      shotObject.transform.Rotate(0f, 0f, Random.Range(-controller.enemyStats.spreadOfShots, controller.enemyStats.spreadOfShots));
      controller.enemyStats.timeOfLastShot = Time.time;
    }
  }
```

----

## Decision

```
[CreateAssetMenu(menuName = "PluggableAI/Decisions/PlayerDetected")]
public class PlayerDetectedDecision : Decision
{
  public override bool Decide(StateController controller)
  {
    return PlayerDetected(controller);
  }

  private bool PlayerDetected(StateController controller)
  {
    if (!DetectionUtils.IsTargetDetected(controller.transform)) return false;
    Debug.unityLogger.Log("Player was detected.");
    return true;
  }
}
```

----

## Transition

```
[System.Serializable]
public class Transition
  {
    public Decision decision;
    public State trueState;
    public State falseState;
  }
```

----

## State

```
[CreateAssetMenu (menuName = "PluggableAI/State")]
public class State : ScriptableObject
{
  public Action[] actions;
  public Transition[] transitions;
  public Color sceneGizmoColor = Color.gray;

  public void UpdateState(StateController controller)
  {
    DoActions(controller);
    CheckTransition(controller);
  }

  private void DoActions(StateController controller)
  {
    foreach (Action action in actions)
    {
      action.Act(controller);
    }
  }

  private void CheckTransition(StateController controller)
  {
    foreach (Transition transition in transitions)
    {
      bool decisionSucceeded = transition.decision.Decide(controller);
      controller.TransitionToState(decisionSucceeded ? transition.trueState : transition.falseState);
    }
  }
}
```

----

![remain-state](images/remain-state.png)

----

![patrolling-state](images/patrol-state.png)

----

![shooting-state](images/shoot-state.png)

----

![state-controller](images/state-controller.png)

----

![raven](images/raven.gif)

----



## Additional Resources

Unity Pluggable AI With Scriptable Objects Tutorial:
https://learn.unity.com/tutorial/5c515373edbc2a001fd5c79d


https://dennis-stepp.com/

Twitter: @destepp

----

You have a story.
<br>
Go forth and **share it**.