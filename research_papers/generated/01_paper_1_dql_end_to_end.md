# Paper 1: Deep Q-Learning as an End-to-End Solver for Time Series

## 1. Abstract & Executive Summary
This paper conceptualizes Deep Q-Learning (DQL) not just as an optimizer that runs on top of pre-computed technical indicators (like MACD, RSI, or Bollinger Bands), but as an **end-to-end continuous solver** that learns directly from the raw financial time series. By plugging the time series matrix directly into recurrent neural network layers (LSTMs or GRUs), the agent learns specialized, non-linear feature maps uniquely tailored to maximize trading rewards, theoretically bypassing the noise and lag of human-engineered features.

## 2. Core Methodology
- **State Representation ($S_t$):** Instead of a static vector of indicators, the state is a sequential sliding lookback window of raw tick/bar data (e.g., the last 60 time steps).
- **Network Architecture:** The DQL's Q-network begins with a recurrent layer (LSTM/GRU) or a 1D Convolutional Neural Network (CNN) to encode the temporal dependencies and multi-cycle lags. The condensed output of this sequence encoding is passed into Dense feed-forward layers to output actionable Q-values (Buy, Sell, Hold).
- **Reward Function ($R_t$):** Operates on realistic trading metrics, typically maximizing the Differential Sharpe Ratio or Net PnL penalized by transaction costs and slippage.

## 3. Core Statistical Assumptions & Concept Check
1. **The Markov Property in Time Series:** 
   Reinforcement Learning mathematically requires the environment to be a Markov Decision Process (MDP) — assuming the *current state* contains all necessary information to predict the optimal future action. True financial markets severely violate the strict Markov property due to long-range memory and macro super-cycles. 
   *How the paper bridges this:* By utilizing Sequence Models (LSTMs) that contain internal "memory cells," combined with sliding sequences, the algorithm artificially constructs a "pseudo-Markov" embedded state capable of retaining historical variance.
2. **Stationarity vs. Non-Stationarity:** 
   Raw nominal prices are non-stationary (they drift without returning to a fixed mean). An end-to-end solver mapping raw prices ($SPY at $500 vs $200) will inevitably fail during Out-of-Sample testing because the neural weights were activated on specific integer boundaries. 
   *Statistical Requirement:* The input series must be rigorously normalized, scaled, or preferably fractionally differenced, to achieve stationarity while retaining maximum structural memory prior to LSTM ingestion.

## 4. Python Implementation Theory (PyTorch)
If we implemented this inside our Antigravity architecture instead of vanilla MLPs, our `DQN` class in `src/dqn_agent.py` would pivot to a sequence-to-vector formulation:

```python
import torch
import torch.nn as nn

class EndToEndDQN(nn.Module):
    def __init__(self, input_features, hidden_size, action_dim):
        super(EndToEndDQN, self).__init__()
        # LSTM replaces standard Dense layers; acts as an automatic time-series extractor
        self.lstm = nn.LSTM(input_size=input_features, hidden_size=hidden_size, batch_first=True)
        # Feed-forward layers convert the extracted temporal embedding into Q-values
        self.fc1 = nn.Linear(hidden_size, 64)
        self.relu = nn.ReLU()
        self.fc2 = nn.Linear(64, action_dim)
        
    def forward(self, x):
        # x shape expected: (batch_size, sequence_lookback_length, features)
        lstm_out, (h_n, c_n) = self.lstm(x)
        # Extract the sequence embedding from the final time step
        last_time_step_out = lstm_out[:, -1, :] 
        q_values = self.fc2(self.relu(self.fc1(last_time_step_out)))
        return q_values
```

## 5. Potential Pitfalls for Project Antigravity
- **Extreme Computation Latency:** Training an LSTM inside a highly iterative Reinforcement Learning loop is severely computationally expensive. Calculating backpropagation-through-time (BPTT) thousands of times per epoch during network replay demands heavy GPU acceleration.
- **Catastrophic Forgetting:** End-to-end architectures can suffer from "catastrophic forgetting"—overwriting historical regime intelligence (e.g., a 2008 crash) when excessively trained on recent, low-volatility bullish sequences.
