# Building the Model Service implementation

## Prerequisites

This part of the tutorial is a continuation of the [Simpl Games API Setup](./games-api.md).  Please complete that section before continuing.  Note, this tutorial assumes you are going through this guide on the same Vagrant box you provisioned in the previous step.

!!! note
    This tutorial assumes you still have Games API service running (http://localhost:8100/).  Please open up an additional terminal window before continuing.

## Installation

First, log into Vagrant and create a new virtualenv called 'calc-model':

```bash
$ vagrant ssh
$ mkvirtualenv calc-model
```

Install Django

```bash
$ pip install Django==1.11.11
```

Change to the projects folder:

```bash
$ cd projects
```

Create a Django project folder and rename it to serve as a git repository

```bash
$ django-admin startproject calc_model
$ mv calc_model calc-model
```

Change to the project folder:

```bash
$ cd calc-model
$ add2virtualenv .
```

Create a `requirements.txt` file that installs the simpl-modelservice and unit testing apps:

```
https://github.com:simplworld/simpl-modelservice/repository/archive.zip

# tests
pytest==3.1.3
pytest-cov==2.5.1
pytest-django==3.1.2
django-test-plus==1.0.22
```

Install these requirements:

```bash
$ PIP_PROCESS_DEPENDENCY_LINKS=1 pip install -r requirements.txt
```

Please note, if `DJANGO_SETTINGS_MODULE` is leftover from a previous session, you may need to unset it:

```bash
$ unset DJANGO_SETTINGS_MODULE
```

Create a django app that will contain your game logic:

```bash
$ ./manage.py startapp game
```

Add the following to your `INSTALLED_APPS` in `calc_model/settings.py`:

```python

INSTALLED_APPS += [
    ...

    'modelservice',
    'rest_framework',

    'game',
]

CALLBACK_URL = os.environ.get('CALLBACK_URL', 'http://{hostname}:{port}/callback')

SIMPL_GAMES_URL = os.environ.get('SIMPL_GAMES_URL', 'http://localhost:8100/apis')

SIMPL_GAMES_AUTH = ('simpl@simpl.world', 'simpl')

ROOT_TOPIC = 'world.simpl.sims.calc'
```

It's highly recommended that you set a `'users'` cache. Since the modelservice will run single-threaded, you can take advantage of the `locmem` backend:

```
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.dummy.DummyCache',
    },
    'users': {
        'BACKEND': 'django.core.cache.backends.locmem.LocMemCache',
        'LOCATION': 'users',
    }
}
```

## Implementation

For simplicity, we're going to create a single player Game in which each player has a Scenario that can advance multiple periods.

In your `game` app module, define our model in `model.py`:

```
class Model(object):
    """
    The model adds an operand to the previous total and returns the result.
    """

    def step(self, operand, prev_total=0.0):
        """
        Parameters:
            operand - current period's decision
            prev_total - the calculated total from the previous period
        Returns new total
        """
        return operand + prev_total
```


Create a 'tests' folder in the `game` folder and add an empty `__init__.py` file.

Add a model unit test `tests/test_model.py`:

```
import pytest
from test_plus.test import TestCase

from game.model import Model


class ModelTestCase(TestCase):
    def setUp(self):
        self.m = Model()

    def test_create(self):
        m = Model()
        self.assertNotEqual(m, None)

    def test_first_step(self):
        m = Model()
        total = m.step(5)
        self.assertEquals(total, 5)

    def test_increase_step(self):
        m = Model()
        total = m.step(5, 3)
        self.assertEquals(total, 8)

    def test_decrease_step(self):
        m = Model()
        total = m.step(5, -2.5)
        self.assertEquals(total, 2.5)
```

Run your unit test:

```
$ export DJANGO_SETTINGS_MODULE=calc_model.settings
$ py.test
```

Create a management command that will create your game and initialize it with one run, a leader and 2 players.

Create a 'management' folder in the `game` folder and add an empty `__init__.py` file.

Create a 'commands' folder in the `management folder  and add an empty `__init__.py` file.

Finally, create a `create_default_env.py` script in the 'commands' folder containing this code:

```
import djclick as click

from modelservice.simpl import games_client
from modelservice.utils.asyncio import coro


def echo(text, value):
    click.echo(
        click.style(text, fg='green') + '{0}'.format(value)
    )


async def delete_default_run(api_session):
    """ Delete default Run """
    echo('Resetting the Calc game default run...', ' done')
    game = await api_session.games.get_or_create(slug='calc')
    runs = await api_session.runs.filter(game=game.id)
    for run in runs:
        if run.name == 'default':
            await api_session.runs.delete(run.id)


@click.command()
@click.option('--reset', default=False, is_flag=True,
              help="Delete default game run and recreate it from scratch")
@coro
async def command(reset):
    """
    Create and initialize Calc game.
    Create a "default" Calc run.
    Set the run phase to "Play".
    Add 1 leader ("leader") to the run
    Add 2 players ("s1", "s2") to the run.
    Add a scenario and period 1 for each player.
    """

    async with games_client as api_session:

        # Handle resetting the game
        if reset:
            if click.confirm(
                    'Are you sure you want to delete the default game run and recreate from scratch?'):
                await delete_default_run(api_session)

        # Create a Game
        game = await api_session.games.get_or_create(
            name='Calc',
            slug='calc'
        )
        echo('getting or creating game: ', game.name)

        # Create required Roles ("Calculator")
        playerRole = await api_session.roles.get_or_create(
            game=game.id,
            name='Calculator',
        )
        echo('getting or creating role: ', playerRole.name)

        # Create game Phases ("Play")
        playPhase = await api_session.phases.get_or_create(
            game=game.id,
            name='Play',
            order=1,
        )
        echo('getting or creating phase: ', playPhase.name)

        # Add run with 2 players ready to play
        run = await add_run(game, 'default', 2, playerRole, playPhase, api_session)

        echo('Completed setting up run: id=', run.id)


async def add_run(game, run_name, user_count, role, phase, api_session):
    # Create or get the Run
    run = await api_session.runs.get_or_create(
        game=game.id,
        name=run_name,
    )
    echo('getting or creating run: ', run.name)

    # Set run to phase
    run.phase = phase.id
    await run.save()
    echo('setting run to phase: ', phase.name)

    fac_user = await api_session.users.get_or_create(
        password='leader',
        first_name='CALC',
        last_name='Leader',
        email='leader@calc.edu',
    )
    echo('getting or creating user: ', fac_user.email)

    fac_runuser = await api_session.runusers.get_or_create(
        user=fac_user.id,
        run=run.id,
        leader=True,
    )
    echo('getting or creating leader runuser for user: ', fac_user.email)

    for n in range(0, user_count):
        user_number = n + 1
        # Add player to run
        await add_player(user_number, run, role, api_session)

    return run


async def add_player(user_number, run, role, api_session):
    """Add player with name based on user_number to run with role"""

    username = 's{0}'.format(user_number)
    first_name = 'Student{0}'.format(user_number)
    email = '{0}@calc.edu'.format(username)

    user = await api_session.users.get_or_create(
        password=username,
        first_name=first_name,
        last_name='User',
        email=email,
    )
    echo('getting or creating user: ', user.email)

    runuser = await api_session.runusers.get_or_create(
        user=user.id,
        run=run.id,
        role=role.id,
    )
    echo('getting or creating runuser for user: ', user.email)

    await add_runuser_scenario(runuser, api_session)


async def add_runuser_scenario(runuser, api_session):
    """Add a scenario named 'Scenario 1' to the runuser"""

    scenario = await api_session.scenarios.get_or_create(
        runuser=runuser.id,
        name='Scenario 1',
    )
    click.echo('getting or creating runuser {} scenario: {}'.format(
        runuser.id,
        scenario.id))

    period = await api_session.periods.get_or_create(
        scenario=scenario.id,
        order=1,
    )
    click.echo('getting or creating runuser {} period 1 for scenario: {}'.format(
        runuser.id,
        scenario.id))

```

Run your command:

```
$ export DJANGO_SETTINGS_MODULE=calc_model.settings
$ ./manage.py create_default_env
```

Every player's move will be a `Decision` made in the current `Period`, the model will produce a `Result` for the current `Period`, and the `Scenario` will step to the next `Period`.

In your `game` app module, create a file called `runmodel.py`. Add `save_decision` and `step_scenario` functions that performs these functions:

```
from modelservice.simpl import games_client
from .model import Model


async def save_decision(period_id, role_id, decision):
    # add decision to period
    decision = await games_client.decisions.get_or_create(
        period=period_id,
        name='decision',
        role=role_id,
        data={"operand": decision}
    )
    return decision


async def step_scenario(scenario_id, role_id):
    """
    Step the scenario's current period
    """
    periods = await games_client.periods.filter(scenario=scenario_id)
    period_count = len(periods)
    period = periods[period_count - 1]

    operand = 0.0
    period_decisions = await games_client.decisions.filter(period=period.id)
    if len(period_decisions) > 0:
        operand = float(period_decisions[0].data["operand"])

    prev_total = 0.0
    if period_count > 1:
        prev_period = periods[period_count - 2]
        prev_period_results = \
            await games_client.results.filter(period=prev_period.id)
        if len(prev_period_results) > 0:
            prev_total = float(prev_period_results[0].data["total"])

    # step model
    model = Model()
    total = model.step(operand, prev_total)
    data = {"total": total}

    result = await games_client.results.get_or_create(
        period=period.id,
        name='results',
        role=role_id,
        data=data
    )

    # prepare for next step by adding a new period
    next_period_order = period.order + 1
    next_period = await games_client.periods.get_or_create(
        scenario=scenario_id,
        order=next_period_order,
    )
    await next_period.save()

    return next_period.id
```


In your `game` app module, create a file called `games.py` with the following content:

```
from modelservice.games import Period, Game
from modelservice.games import subscribe, register

from .runmodel import step_scenario, save_decision


class CalcPeriod(Period):
    @subscribe
    async def submit_decision(self, operand, **kwargs):
        """
        Receives the operand played and stores as a ``Decision`` then
        steps the model saving the ``Result``. A new ``Period`` is added to
        scenario in preparation for the next decision.
        """
        # Call will prefix the ROOT_TOPIC
        # "world.simpl.sims.calc.model.period.1.submit_decision"

        for k in kwargs:
            self.session.log.info("submit_decision: Key: {}".format(k))

        user = kwargs['user']
        runuser = self.game.get_scope('runuser', user.runuser.pk)
        role = runuser.role

        await save_decision(self.pk, role.pk, operand)
        self.session.log.info("submit_decision: saved decision")

        await step_scenario(self.scenario.pk, role.pk)
        self.session.log.info("submit_decision: stepped scenario")


Game.register('calc', [
    CalcPeriod,
])
```
**NOTE:** if you
want to use a filename other than `games.py` you must ensure the file is imported
somewhere, usually in a `__init__.py` somewhere for the `@game` decorator to find
and register your game into the system.


You can start your model service by running:

```bash
$ export DJANGO_SETTINGS_MODULE=calc_model.settings
$ ./manage.py run_modelservice
```

By default the service will bind to `0.0.0.0:8080`.

This concludes the tutorial on Model Service. A completed example implementation is available at `github.com:simplworld/simpl-calc-model.git`
that uses the game slug `simpl-calc`.

You can now head over to the [Frontend tutorial](./frontend.md).
