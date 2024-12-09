# Police_Data_Dashboard
from dash import Dash, dcc, html, Input, Output
import pandas as pd
import plotly.express as px
import datetime

# Load and preprocess the data
data_path = "Police_Bulk_Data_2014_20241027.csv"
police_data = pd.read_csv(data_path)
police_data['offensedate'] = pd.to_datetime(police_data['offensedate'], errors='coerce')

# Create the Dash app
app = Dash(__name__)

# Layout
app.layout = html.Div([
    html.H1("Interactive Police Data Dashboard", style={'textAlign': 'center'}),

    # Filters
    html.Div([
        html.Label("Select Offense Status:"),
        dcc.Dropdown(
            id='status-dropdown',
            options=[{'label': status, 'value': status} for status in police_data['offensestatus'].unique()],
            value=police_data['offensestatus'].unique().tolist(),
            multi=True
        ),
        html.Label("Select Date Range:"),
        dcc.DatePickerRange(
            id='date-picker',
            start_date=police_data['offensedate'].min(),
            end_date=police_data['offensedate'].max(),
            display_format='YYYY-MM-DD'
        ),
    ], style={'margin': '20px'}),

    # Visualizations
    html.Div([
        html.H3("Top 10 Most Common Offenses"),
        dcc.Graph(id='offense-bar-chart'),
    ]),
    html.Div([
        html.H3("Monthly Offense Trends"),
        dcc.Graph(id='offense-trend-chart'),
    ]),
    html.Div([
        html.H3("Offenses by City"),
        dcc.Graph(id='offense-city-chart'),
    ])
])

# Callbacks for interactivity
@app.callback(
    [Output('offense-bar-chart', 'figure'),
     Output('offense-trend-chart', 'figure'),
     Output('offense-city-chart', 'figure')],
    [Input('status-dropdown', 'value'),
     Input('date-picker', 'start_date'),
     Input('date-picker', 'end_date')]
)
def update_charts(selected_status, start_date, end_date):
    # Filter data based on selections
    filtered_data = police_data[
        (police_data['offensestatus'].isin(selected_status)) &
        (police_data['offensedate'] >= pd.to_datetime(start_date)) &
        (police_data['offensedate'] <= pd.to_datetime(end_date))
    ]

    # Chart 1: Top 10 Most Common Offenses
    top_offenses = filtered_data['offensedescription'].value_counts().head(10)
    fig1 = px.bar(
        top_offenses,
        x=top_offenses.index,
        y=top_offenses.values,
        labels={'x': 'Offense Description', 'y': 'Number of Offenses'},
        title="Top 10 Most Common Offenses"
    )

    # Chart 2: Monthly Offense Trends
    monthly_trends = filtered_data.groupby(filtered_data['offensedate'].dt.to_period("M")).size()
    monthly_trends.index = monthly_trends.index.to_timestamp()
    fig2 = px.line(
        monthly_trends,
        x=monthly_trends.index,
        y=monthly_trends.values,
        labels={'x': 'Date', 'y': 'Number of Offenses'},
        title="Monthly Offense Trends"
    )

    # Chart 3: Offenses by City
    city_data = (
        filtered_data[['offensecity', 'offensestate']]
        .dropna()
        .value_counts()
        .reset_index(name='Offense Count')
        .rename(columns={'offensecity': 'City', 'offensestate': 'State'})
        .head(10)
    )
    fig3 = px.bar(
        city_data,
        x='City',
        y='Offense Count',
        title="Offenses by City",
        text='Offense Count'
    )

    return fig1, fig2, fig3

# Run the app
if __name__ == '__main__':
    app.run_server(debug=True)
