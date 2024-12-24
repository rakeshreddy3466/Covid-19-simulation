import pandas as pd
import numpy as np
from sim_parameters import TRANSITION_PROBS, HOLDING_TIMES
import helper

def create_timeseries(sample_population_df, start_date, end_date):

    # Private function to simulate the Markov chain
    def _simulate_markov_chain(num_days, age_group):
        state_history, previous_state_history, days_in_current_state = [], [], []
        current_state = 'H'
        prior_state = 'H'
        remaining_days_in_state = 0

        # Loop through each day of the simulation
        for _ in range(num_days):
            state_history.append(current_state)
            previous_state_history.append(prior_state)
            days_in_current_state.append(remaining_days_in_state)

            if remaining_days_in_state <= 1:
                prior_state = current_state
                current_state = np.random.choice(
                    list(TRANSITION_PROBS[age_group][current_state].keys()),
                    p=list(TRANSITION_PROBS[age_group][current_state].values())
                )
                remaining_days_in_state = HOLDING_TIMES[age_group][current_state]
            else:
                remaining_days_in_state -= 1

        return state_history, previous_state_history, days_in_current_state

    # Create a empty df with the required columns - timeseries_df
    timeseries_columns = ['person_id', 'age_group_name', 'country', 'date', 'state', 'staying_days', 'prev_state']
    timeseries_df = pd.DataFrame(columns=timeseries_columns)

    # List of dates between start_date and end_date
    date_range = pd.date_range(start=start_date, end=end_date)
    total_days = len(date_range)

    # Loop each person in the sample_population_df to simulate the Markov chain
    for idx, person in sample_population_df.iterrows():

        # Each person's timeseries data
        person_timeseries = pd.DataFrame()
        person_timeseries["date"] = date_range
        person_timeseries["person_id"] = person["person_id"]
        age_group = person["age_group_name"]
        person_timeseries['age_group_name'] = age_group
        person_timeseries['country'] = person['country']

        # The Actual Markov Chain Simulation
        state_series, prev_state_series, days_series = _simulate_markov_chain(total_days, age_group)

        # Add sim data to the person's timeseries DataFrame
        person_timeseries["state"] = state_series
        person_timeseries["staying_days"] = days_series
        person_timeseries["prev_state"] = prev_state_series

        # Append this person's data to the main timeseries_df
        timeseries_df = pd.concat([timeseries_df, person_timeseries], ignore_index=True)

    return timeseries_df


def run(countries_csv_name, countries, sample_ratio, start_date, end_date):

    df = pd.read_csv(countries_csv_name)

    #Generating a sample population
    filtered_countries = df[df['country'].isin(countries)]
    sample_pop_list = list()
    p_id = 0

    for i, row in filtered_countries.iterrows():
        sample_size = int(row['population'] / sample_ratio) # Total sample size/1e6
        age_group_pop_size = {
            'less_5': int((row['less_5'] * sample_size) / 100),
            '5_to_14': int((row['5_to_14'] * sample_size) / 100),
            '15_to_24': int((row['15_to_24'] * sample_size) / 100),
            '25_to_64': int((row['25_to_64'] * sample_size) / 100),
            'over_65': int((row['over_65'] * sample_size) / 100)
        }
        for each_age_grp, size in age_group_pop_size.items():
            for _ in range(size):
                p_id += 1
                sample_pop_list.append({
                    'person_id': p_id,
                    'age_group_name': each_age_grp,
                    'country': row['country']
                })
        
        sample_population = pd.DataFrame(sample_pop_list, columns=['person_id', 'age_group_name', 'country'])

    #Creating timeseries
    timeseries = create_timeseries(sample_population, start_date, end_date)
    timeseries.to_csv("a2-covid-simuated-timeseries.csv")

    # Group by 'country', 'date', and 'state' and count occurrences of each state
    grouped_timeseries = timeseries.groupby(['country', 'date', 'state']).size().reset_index(name='count')

    # Pivot the table to have states as columns, and fill missing values with 0
    summarized_timeseries = grouped_timeseries.pivot_table(index=['date', 'country'], 
                                                       columns='state', 
                                                       values='count', 
                                                       fill_value=0)

    # Ensure the data type is integer
    summarized_timeseries = summarized_timeseries.astype(int)
    summarized_timeseries.to_csv('a2-covid-summary-timeseries.csv')


    helper.create_plot('a2-covid-summary-timeseries.csv', countries)

    return summarized_timeseries
