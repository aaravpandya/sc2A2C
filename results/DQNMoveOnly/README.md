# DQNMoveOnly Results

## Model Details

### Inputs/Outputs
This model operates on reduced state and action spaces, processing only the `player_relative` screen feature layer, and outputting a spatial coordinate meant to be used as an argument to `Move_screen("now", ...)`.

### Estimator
A deep Q network (DQN) is used to model the value (Q) of each action conditioned on the state. The value represents the expected total reward due to selecting an action, and is equal to the expected immediate reward plus the best value achievable from the state the environment will transition to as a result of the action - discounted multiplicatively by `discount_factor`.

The network first embeds the feature layer into a continuous space by one-hot encoding in the channel dimension and running that through a 1x1 convolutional layer. This is followed by a single convolutional layer with 16 3x3 filters, a stride of 1, ReLU activation, and "SAME" padding in order to preserve the spatial dimension. This is done so that the output layer can be convolutional, sparsely connected to the previous convolutional layer with a single 1x1 filter. The spatial coordinates of each output unit represent the (x, y) coordinates of a `Move_screen` action, while the numbers produced by the linear activations represent the Q values associated with those coordinates.

### Policy
At each step, with probability `epsilon_min` a random action is chosen, otherwise the action having the maximum Q-value according to the DQN is chosen. The `epsilon_min` used by the learned policy need not be that used during training, and may be 0.

### Training Procedure
An epsilon-greedy strategy is used to explore the state/action space. The probability of selecting a random action, epsilon, is annealed linearly from a maximum value `epsilon_max` to a minimum value `epsilon_min` over a number of steps equaling `epsilon_decay_steps`. When selecting actions nonrandomly, the agent acts according to its policy, selecting the action with the highest estimated value.

After every `train_frequency` steps, the online DQN (the network used to inform the policy) has its weights updated using gradient descent, where the target Q given an action is the immediate reward plus the maximum estimated Q in the resulting state.

A separate copy of the network is used to calculate the values of states used as a term in the target Q's when training the online DQN. Periodically (every `target_update_frequency` steps), the weights of the online DQN are copied over to the target DQN. This is to stabilize the values predicted, and to prevent feedback loops leading to large overestimations.

The action-reward-next_state data used as inputs during the gradient updates to the online DQN are stored in a memory buffer. During training, a random sample of `batch_size` experiences are sampled from the buffer. This Experience Replay method is meant to decorrelate the experiences the value estimator trains on, since those generated by the training (epsilon greedy) policy will be highly autocorrelated. The Experience Replay buffer stores a maximum of `max_memory` experiences in a double-ended queue.

## Results
<table align="center">
  <tr>
    <td align="center"></td>
    <td align="center">Test Score</td>
    <td align="center">Training Episodes</td>
    <td align="center">Notes</td>

  </tr>
  <tr>
    <td align="center">
      MoveToBeacon<br>
      (run 1)<br>
      <a href="https://drive.google.com/file/d/18BTNB8T2JHdEyw_Fg34lrJcsA0pSst5L/view?usp=sharing">checkpoint</a>
    </td>
    <td align="center">
      ~1 (mean)<br>
      7 (max) 
    </td>
    <td align="center">
      1,000 (~120,000 steps)
    </td>
    <td align="left">
      <ul>
        <li><code>step_mul</code> flag set to 16 during training.</li>
        <li>failed to learn an optimal policy, gets stuck at edge of beacon.</li>
        <li>test score evaluated over 100 episodes with epsilon=0.</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td align="center">
      MoveToBeacon<br>
      (run 2)<br>
      <a href="https://drive.google.com/open?id=1GAoxY1fEkDH8LO2_IwigYpTmsuGstyxn">checkpoint</a>
    </td>
    <td align="center">
      <strong>~20</strong> (mean)<br>
      <strong>23</strong> (max)
    </td>
    <td align="center">
      1,000 (~120,000 steps)
    </td>
    <td align="left">
      <ul>
        <li><code>step_mul</code> flag set to 16 during training.</li>
        <li><code>step_mul</code> flag set to 16 during testing.</li>
        <li>test score evaluated over 100 episodes with epsilon=0.</li>
        <li>policy does not generalize well to a <code>step_mul</code> of 8, agent gets stuck at edge of some beacon positions.</li>
        <li>test score with <code>step_mul</code>=8: ~13 (mean), 21 (max).</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td align="center">
      CollectMineralShards<br>
      (run 1)<br>
      <a href="https://drive.google.com/open?id=1feaeCrnAfU7n5_EEcqqB0s6qS5HED5bB">checkpoint</a>
    </td>
    <td align="center">
      <strong>~75</strong> (mean)<br>
      <strong>99</strong> (max)
    </td>
    <td align="center">
      1,000 (~120,000 steps)
    </td>
    <td align="left">
      <ul>
        <li>test score evaluated over 100 episodes with epsilon=0.</li>
        <li>has learned to collect roughly from left to right, with inefficient vertical oscillations.</li>
      </ul>
    </td>
  </tr>
</table>

## Training Notes/Caveats
* The Experience Replay buffer does not persist over multiple runs, and is repopulated from scratch each time.
* The target network is updated at the beginning of each new run, even when restoring the online network from a checkpoint.