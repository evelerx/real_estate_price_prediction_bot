# real_estate_price_prediction_bot
import streamlit as st
import pandas as pd
import requests
from sklearn.ensemble import RandomForestRegressor
from sklearn.preprocessing import StandardScaler
from io import BytesIO
import firebase_admin
from firebase_admin import credentials
from datetime import datetime

# Replace with your Firebase credentials file content
cred = credentials.Certificate({
    "type": "service_account",
    "project_id": "YOUR_FIREBASE_PROJECT_ID",
    "private_key_id": "YOUR_PRIVATE_KEY_ID",
    "private_key": "-----BEGIN PRIVATE KEY-----\nYOUR_PRIVATE_KEY\n-----END PRIVATE KEY-----\n",
    "client_email": "YOUR_CLIENT_EMAIL",
    "client_id": "YOUR_CLIENT_ID",
    "auth_uri": "https://accounts.google.com/o/oauth2/auth",
    "token_uri": "https://oauth2.googleapis.com/token",
    "auth_provider_x509_cert_url": "https://www.googleapis.com/oauth2/v1/certs",
    "client_x509_cert_url": "YOUR_CLIENT_CERT_URL",
})
firebase_admin.initialize_app(cred)

# Mock pre-trained model (replace with your trained model)
model = RandomForestRegressor()
scaler = StandardScaler()

# API functions for real-time data fetching

def get_zillow_data(location):
    zillow_api_key = "YOUR_ZILLOW_API_KEY"
    zillow_url = f"https://api.zillow.com/webservice/GetZestimate.htm?address={location}&apikey={zillow_api_key}"
    response = requests.get(zillow_url)
    
    if response.status_code == 200:
        data = response.json()
        price_trend = data['response']['zestimate']['value']
        return {"price_trend": price_trend}
    else:
        return {"error": "Unable to fetch Zillow data"}

def get_population_data(location):
    census_api_key = "YOUR_CENSUS_API_KEY"
    census_url = f"https://api.census.gov/data/2019/acs/acs5?get=NAME,B01003_001E&for=county:*&in=state:{location}&key={census_api_key}"
    
    response = requests.get(census_url)
    if response.status_code == 200:
        data = response.json()
        population_growth = data[1][1]  # Extracting population growth
        return {"population_growth": population_growth}
    else:
        return {"error": "Unable to fetch population data"}

def get_interest_rate(country):
    if country.lower() == "us":
        fredd_api_key = "YOUR_FRED_API_KEY"
        fred_url = f"https://api.stlouisfed.org/fred/series/observations?series_id=IR14250&api_key={fredd_api_key}&file_type=json"
        
        response = requests.get(fred_url)
        if response.status_code == 200:
            data = response.json()
            interest_rate = data['observations'][0]['value']  # Extract the interest rate value
            return {"interest_rate": float(interest_rate)}
        else:
            return {"error": "Unable to fetch interest rate"}
    else:
        return {"error": "Interest rate data unavailable for this country"}

# Initialize Streamlit
st.title("Real Estate Price Prediction for India and Global Markets")

# Form for user input
with st.form(key='property_form'):
    st.subheader("Enter Property Details")

    # Input for multiple properties
    property_name = st.text_input("Property Name")
    size = st.number_input("Size (sq ft)", min_value=100)
    bedrooms = st.number_input("Number of Bedrooms (for Residential only)", min_value=0, step=1)
    bathrooms = st.number_input("Number of Bathrooms (for Residential only)", min_value=0, step=1)
    
    property_type = st.selectbox(
        "Property Type",
        options=["Residential", "Commercial", "Industrial", "Land"]
    )
    
    state = st.text_input("State")
    country = st.text_input("Country")
    country_flag = st.text_input("Country Flag (Emoji)")

    price = st.number_input("Current Price", min_value=1000)
    years = st.number_input("Predict price after how many years?", min_value=1, value=4)
    
    submit_button = st.form_submit_button("Add Property")

    if submit_button:
        property_data = {
            "property_name": property_name,
            "size": size,
            "bedrooms": bedrooms,
            "bathrooms": bathrooms,
            "property_type": property_type,
            "state": state,
            "country": country,
            "country_flag": country_flag,
            "price": price,
            "years": years
        }
        if 'properties' not in st.session_state:
            st.session_state['properties'] = []
        st.session_state['properties'].append(property_data)
        st.success(f"Property '{property_name}' added!")

# Display added properties
if len(st.session_state['properties']) > 0:
    st.write("### Properties Added")
    properties_df = pd.DataFrame(st.session_state['properties'])
    st.dataframe(properties_df)

# Predict prices for all properties
if st.button("Predict Prices for All Properties"):
    results = []
    for property_data in st.session_state['properties']:
        location = property_data['state'] + ", " + property_data['country']

        # Fetch data from APIs
        zillow_data = get_zillow_data(location)
        population_data = get_population_data(location)
        interest_rate = get_interest_rate(property_data['country'])

        if "error" not in zillow_data and "error" not in population_data and "error" not in interest_rate:
            price_trend = zillow_data['price_trend'] / 100
            population_growth = population_data['population_growth']
            current_interest_rate = interest_rate['interest_rate']

            # Adjust model based on property type
            if property_data['property_type'] == 'Commercial':
                price_trend *= 1.2
            elif property_data['property_type'] == 'Industrial':
                price_trend *= 0.8
            elif property_data['property_type'] == 'Land':
                price_trend *= 0.5

            # Prepare input data for prediction
            input_data = pd.DataFrame({
                'size': [property_data['size']],
                'bedrooms': [property_data['bedrooms']] if property_data['property_type'] == 'Residential' else [0],
                'bathrooms': [property_data['bathrooms']] if property_data['property_type'] == 'Residential' else [0],
                'price_trend': [price_trend],
                'population_growth': [population_growth],
                'interest_rate': [current_interest_rate]
            })

            # Feature scaling
            scaled_input = scaler.transform(input_data)

            # Make prediction
            predicted_price = model.predict(scaled_input)[0]

            # Predict future price
            years = property_data['years']
            future_price = predicted_price * (1 + price_trend) ** years

            # Store the result
            result = {
                "property_name": property_data['property_name'],
                "size": property_data['size'],
                "bedrooms": property_data['bedrooms'] if property_data['property_type'] == 'Residential' else 0,
                "bathrooms": property_data['bathrooms'] if property_data['property_type'] == 'Residential' else 0,
                "property_type": property_data['property_type'],
                "price": f"${property_data['price']:,.2f}",
                "years": property_data['years'],
                "future_price": f"${future_price:,.2f}",
                "price_trend": price_trend * 100,
                "population_growth": population_growth,
                "interest_rate": current_interest_rate,
                "state": property_data['state'],
                "country": property_data['country'],
                "country_flag": property_data['country_flag']
            }
            results.append(result)

    results_df = pd.DataFrame(results)
    st.write("### Prediction Results")
    st.dataframe(results_df)

    # Export to Excel
    excel_output = BytesIO()
    with pd.ExcelWriter(excel_output, engine='xlsxwriter') as writer:
        results_df.to_excel(writer, index=False, sheet_name="Predictions")
    excel_output.seek(0)

    timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
    file_name = f"real_estate_predictions_{timestamp}.xlsx"

    st.download_button(label="Download Predictions as Excel", data=excel_output, file_name=file_name, mime="application/vnd.openxmlformats-officedocument.spreadsheetml.sheet")

