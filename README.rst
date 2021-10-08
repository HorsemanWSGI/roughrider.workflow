roughrider.workflow
*******************

This package allows to define workflows that consist of a set of states,
transitions between the states. Actions that activate the transitions
can be guarded by constraints.


Example
=======

.. code-block:: python

  from roughrider.predicate import ConstraintError, Validator, Or
  from roughrider.workflow.components import Action, Transition, Transitions
  from roughrider.workflow.workflow import (
      WorkflowItem, WorkflowState, Workflow)


  class Document:
      state = None
      body = ""


  class RoleValidator(Validator):

      def __init__(self, role):
          self.role = role

      def __call__(self, item, role=None, **namespace):
          if role != self.role:
              raise ConstraintError(
                  message=f'Unauthorized. Missing the `{role}` role.')


  class PublicationWorkflow(Workflow):

      class wrapper(WorkflowItem):

          @property
          def state(self):
              return self.workflow.get(self.item.state)

          @state.setter
          def state(self, state):
              self.item.state = state.name


      class states(WorkflowState):
          draft = 'Draft'
          published = 'Published'
          submitted = 'Submitted'


      transitions = Transitions((
          Transition(
              origin=states.draft,
              target=states.published,
              action=Action(
                  'Publish',
                  constraints=[RoleValidator('publisher')]
              )
          ),
          Transition(
              origin=states.published,
              target=states.draft,
              action=Action(
                  'Retract',
                  constraints=[
                      Or((RoleValidator('owner'),
                          RoleValidator('publisher')))
                  ]
              )
          ),
          Transition(
              origin=states.draft,
              target=states.submitted,
              action=Action(
                  'Submit',
                  constraints=[RoleValidator('owner')],
              )
          ),
          Transition(
              origin=states.submitted,
              target=states.published,
              action=Action(
                  'Publish',
                  constraints=[RoleValidator('publisher')],
              )
          )
      ))


  workflow = PublicationWorkflow('draft')  # initial state
  item = Document()
  workflow_item = workflow(item, role='owner')
  workflow_item.transition_to(PublicationWorkflow.states.submitted)
