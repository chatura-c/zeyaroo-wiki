when a client send a inquiry it should show up in the inquiries page.
creator can send

when a client sends a inquiry it should show up in inquiries page. creator should be able send a proposal to the inquiry. this stage may go back and forth between creator and
    the client until they agree on a final proposal. after that creator can click a button right from the proposal page and create a project. all the versions of the proposal
    should be stored as immutable objects so that if ever needed it's possible to look into history. proposals should be handled by a state machine where the status are inquiry -
    when client first sends the inquiry, pending - while it's on creators hand,  negotiating - when creator sends the proposal and it's now on client's hand to accept or suggest
  changes. finalized - when client accepts the proposal--this is when the creator can create a project based on the proposal, canceled - if either party has decided to cancel the
  proposal
