#!/usr/bin/env python
# coding: utf-8

# # Compound Pool Economics - The Graph

# In[1]:


# cadCAD standard dependencies

# cadCAD configuration modules
from cadCAD.configuration.utils import config_sim
from cadCAD.configuration import Experiment

# cadCAD simulation engine modules
from cadCAD.engine import ExecutionMode, ExecutionContext
from cadCAD.engine import Executor

# cadCAD global simulation configuration list
from cadCAD import configs


# In[2]:


# Additional dependencies

# For parsing the data from the API
import json
# For downloading data from API
import requests as req
# For generating random numbers
import math
# For analytics
import pandas as pd
# For visualization
import plotly.express as px
import numpy as np
import datetime


# # Setup / Preparatory Steps
# 
# ## Query the Balancer subgraph for a UNI-BAL pool

# In[3]:


# You can explore the subgraph at https://thegraph.com/hosted-service/subgraph/graphprotocol/compound-v2
API_URI = 'https://api.thegraph.com/subgraphs/name/graphprotocol/compound-v2'

# Query for retrieving the history of swaps on a BAL <> UNI 50-50 pool
GRAPH_QUERY = '''
{
  markets{
        borrowRate
        supplyRate
        totalBorrows
        totalSupply
        exchangeRate
  }
}
'''
'''
    borrowRate
    cash
    collateralFactor
    exchangeRate
    interestRateModelAddress
    name
    reserves
    supplyRate
    symbol
    id
    totalBorrows
    totalSupply
    underlyingAddress
    underlyingName
    underlyingPrice
    underlyingSymbol
    reserveFactor
    underlyingPriceUSD
'''

# Retrieve data from query
JSON = {'query': GRAPH_QUERY}
r = req.post(API_URI, json=JSON)
graph_data = json.loads(r.content)['data']

print("Print first 200 characters of the response")
print(r.text[:200])


# ## Data Wrangle the Data

# In[4]:


raw_df = pd.DataFrame(graph_data['markets'])

raw_df.head(5)


# In[5]:


# Clean the data:
# 1. convert the raw timestamps to Python DateTime objects
# 2. make the token flow values numerical
# 3. order by time
df = (raw_df.assign(totalBorrows=lambda df: pd.to_numeric(df.totalBorrows))
            .assign(totalSupply=lambda df: pd.to_numeric(df.totalSupply))
            .assign(borrowRate=lambda df: pd.to_numeric(df.borrowRate))
            .assign(supplyRate=lambda df: pd.to_numeric(df.supplyRate))
            .assign(exchangeRate=lambda df: pd.to_numeric(df.exchangeRate))
            #.assign(blockTimestamp=lambda df: (pd.to_datetime(df.blockTimestamp, unit='s')))
            .reset_index()
      )

df.head(5)


# # Modelling

# ## 1. State Variables

# In[6]:


initial_state = {
    'lender_APY': 0.0,
    'borrower_rate': 0.0,
    'utilization_rate': 0.0,
    'exchange_rate': 0.0
    #'block_time_stamp': None
}
initial_state


# ## 2. System Parameters

# In[7]:


# Transform the swap history data frame into a {timestep: data} dictionary
# Turning the df into a dictionary form
df_dict = df.to_dict(orient='index')

system_params = {
    'new_df': [df_dict]
    
    # Transaction fees being applied to the input token
    #'exchangeRate': df['exchangeRate']
    #'block_time_stamp': ['blockTimestamp']
}

# Element for timestep = 3


# In[ ]:





# ## 3. Policy Functions

# In[8]:


def p_rates(params, substep, state_history, previous_state):
    """
    Calculate cumulative transaction fees & swaps
    from a swap event
    """
    t = previous_state['timestep']
    
    # Data for this timestep
    ts_data = params['new_df'][t]    
    

    lender_APY = ts_data['supplyRate']
 
    borrower_rate = ts_data['borrowRate']
    
    exchange_rate = ts_data['exchangeRate']
    
    #block_time_stamp = ts_data['blockTimestamp']
    
    total_borrowed = ts_data['totalBorrows']
    TVL = ts_data['totalSupply']

    #utilization_rate = total_borrowed / TVL * 100
    try:
        utilization_rate = pd.to_numeric(total_borrowed) / pd.to_numeric(TVL) * 100
    except ZeroDivisionError:
        utilization_rate = 0
        
    #exchange_rate1 = swap_in * params['exchange_rate']

    return {'lender_APY': lender_APY,
            'borrower_rate': borrower_rate,
            'exchange_rate': exchange_rate,
            'utilization_rate': utilization_rate}
            #'block_time_stamp': block_time_stamp


# In[9]:


print(df.dtypes)


# ## 4. State Update Functions

# In[10]:


def s_lender_APY(params,
                      substep,
                      state_history,
                      previous_state,
                      policy_input):
    value = policy_input['lender_APY']
    return ('lender_APY', value)

def s_borrower_APY(params,
                              substep,
                              state_history,
                              previous_state,
                              policy_input):
    value = policy_input['borrower_rate']
    #fee = policy_input['fee_UNI']
    #value = previous_state['cumulative_fee_UNI'] + fee 
    return ('borrower_rate', value)


def s_utilization_rate(params,
                              substep,
                              state_history,
                              previous_state,
                              policy_input):
    value = policy_input['utilization_rate']
    #fee = policy_input['fee_BAL']
    #value = previous_state['cumulative_fee_BAL'] + fee 
    return ('utilization_rate', value)

def s_exchange_rate(params,
                              substep,
                              state_history,
                              previous_state,
                              policy_input):
    value = policy_input['exchange_rate']
    #fee = policy_input['fee_BAL']
    #value = previous_state['cumulative_fee_BAL'] + fee 
    return ('exchange_rate', value)

'''def s_block_time_stamp(params,
                              substep,
                              state_history,
                              previous_state,
                              policy_input):
    value = policy_input['block_time_stamp']
    #fee = policy_input['fee_BAL']
    #value = previous_state['cumulative_fee_BAL'] + fee 
    return ('block_time_stamp', value)'''


# ## 5. Partial State Update Blocks

# In[11]:


partial_state_update_blocks = [
    {
        'policies': {
            'policy_rates': p_rates
        },
        'variables': {
            's_lender_APY': s_lender_APY,
            's_borrower_rate': s_borrower_APY,
            's_exchange_rate': s_exchange_rate,
            's_utilization_rate': s_utilization_rate
            #'s_block_time_stamp': s_block_time_stamp
        }
    }
]


# # Simulation

# ## 6. Configuration

# In[12]:


sim_config = config_sim({
    "N": 1, # the number of times we'll run the simulation ("Monte Carlo runs")
    "T": range(len(df)), # the number of timesteps the simulation will run for
    "M": system_params # the parameters of the system
})


# In[13]:


del configs[:] # Clear any prior configs


# In[14]:


experiment = Experiment()
experiment.append_configs(
    initial_state = initial_state,
    partial_state_update_blocks = partial_state_update_blocks,
    sim_configs = sim_config
)


# ## 7. Execution

# In[15]:


exec_context = ExecutionContext()
simulation = Executor(exec_context=exec_context, configs=configs)
raw_result, tensor_field, sessions = simulation.execute()


# ## 8. Output Preparation

# In[16]:


simulation_result = pd.DataFrame(raw_result)
simulation_result.head(5)


# ## 9. Analysis

# In[17]:


# Visualize how much transaction fees were paid over time on each token
print("High supply of lenders → Low utilization rate → Lower lender APY")
print("High demand for borrowing → High utilization rate → Higher borrower rates")
graph = px.line(simulation_result,
           x='timestep',
           y=['borrower_rate','lender_APY','exchange_rate'],
              #],
              #, 'utilization_rate'],
           title='Compound Pool Economics',
           facet_row='subset')

graph.update_layout(yaxis=dict(tickformat="%", hoverformat="%.2f%"))
graph1 = graph.update_yaxes(hoverformat=".2%")
graph1


# In[18]:


ur_graph = px.line(simulation_result,
           x='timestep',
           y=['utilization_rate'],
           facet_row='subset')

ur_graph = ur_graph.update_layout(yaxis=dict(tickformat="%", hoverformat="%.2f%"))
ur_graph1 = ur_graph.update_yaxes(hoverformat=".2%")
ur_graph1


# In[ ]:





# In[ ]:


