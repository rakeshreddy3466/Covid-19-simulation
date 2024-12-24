import streamlit as st
import pandas as pd
import assignment2
import matplotlib.pyplot as plt
import io
import matplotlib.dates as md

def create_plot(summarized_timeseries, countries, save_image=False, output_name="covid_simulation_plot.png"):
    summarized_timeseries = summarized_timeseries.reset_index()
    summarized_timeseries['date'] = pd.to_datetime(summarized_timeseries['date'])
    
    expected_columns = ['country', 'date', 'H', 'I', 'S', 'M', 'D']
    summarized_timeseries = summarized_timeseries[expected_columns]
    
    countries_num = len(countries)
    fig, ax = plt.subplots(countries_num, figsize=(16, 9 * countries_num))
    
    if countries_num == 1:
        ax = [ax]
    
    for i, country in enumerate(countries):
        country_data = summarized_timeseries[summarized_timeseries['country'] == country]
        country_data.plot(kind='bar', x='date', stacked=True, width=1,
                          color=['green', 'darkorange', 'indianred', 'lightseagreen', 'slategray'],
                          ax=ax[i])
        ax[i].legend(['Healthy', 'Infected (without symptoms)', 'Infected (with symptoms)', 'Immune', 'Deceased'])
        ax[i].set_title(f"Covid Infection Status in {country}")
        ax[i].set_xlabel("Date")
        ax[i].set_ylabel("Population in Millions")

        selected_dates = country_data['date'].dt.to_period('M').unique()
        ax[i].set_xticks(range(len(selected_dates)))
        ax[i].set_xticklabels(selected_dates.strftime('%b %Y'), rotation=30, horizontalalignment="center")
        ax[i].xaxis.set_major_locator(md.MonthLocator())

    plt.tight_layout()
    
    if save_image:
        fig.savefig(output_name, dpi=300)
    
    image_buffer = io.BytesIO()
    fig.savefig(image_buffer, format='png', dpi=300)
    plt.close(fig)
    image_buffer.seek(0) 
    
    return image_buffer  

# Streamlit UI code
st.title("A3 Test Runner")

sample_ratio = st.number_input("Sample Ratio", value=1000000.00, step=10000.0, format="%.2f")

start_date = st.date_input("Start Date", value=pd.to_datetime('2021-04-01'))
end_date = st.date_input("End Date", value=pd.to_datetime('2022-04-30'))

countries_csv_name = "a2-countries.csv"
df = pd.read_csv(countries_csv_name)
country_list = df['country'].tolist()

selected_countries = st.multiselect("Select Countries", country_list, default=["Afghanistan", "Sweden", "Argentina"])

if st.button("Run"):
    st.write(f"Running with the following settings: Sample Ratio: {sample_ratio}, Start Date: {start_date}, End Date: {end_date}, Selected Countries: {selected_countries}")

    try:
        timeseries_data = assignment2.run(
                                    countries_csv_name='a2-countries.csv',
                                    countries=selected_countries,
                                    sample_ratio=sample_ratio,
                                    start_date=start_date.strftime("%Y-%m-%d"),
                                    end_date=end_date.strftime("%Y-%m-%d"))
        
        plot = create_plot(timeseries_data, selected_countries)
        st.image(plot, caption="COVID-19 Simulation Timeseries", use_column_width=True)

    except Exception as e:
        st.write(f"An error occurred: {e}")
