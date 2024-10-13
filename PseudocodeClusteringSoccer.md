# Extract Data

    ## League - Year
    FUNCTION fetch_player_stats(page)
        SET url TO "API_URL_FOR_PAGE"  # Define the API URL for the specified page
        SET response TO READ url
        SET data TO PARSE_JSON(response)
        
        # Adjusting for the correct response format
        IF 'results' EXISTS IN data THEN
            RETURN data['results']
        ELSE
            PRINT "Unexpected data structure: " + data
            RETURN []
    END FUNCTION
    
    FUNCTION fetch_player_details(player_id)
        SET url TO "url" + player_id
        SET response TO GET(url)
        SET data TO GET("player", response)
    
        FUNCTION calculate_age(birth_timestamp)
            SET today TO CURRENT_TIME()
            SET age TO (today - birth_timestamp) // (365.25 * 24 * 3600)
            RETURN CAST(age AS INTEGER)
        END FUNCTION
    
        FUNCTION convert_value(value)
            RETURN value / 1000000
        END FUNCTION
    
        SET age TO calculate_age(data.get('dateOfBirthTimestamp', 0))
        SET market_value TO convert_value(data.get('proposedMarketValue', 0))
        SET position TO data.get('position', 'N/A')
        SET preferred_foot TO data.get('preferredFoot', 'N/A')
    
        RETURN {
            'MarketValue': market_value,
            'Age': age,
            'Position': position,
            'PreferredFoot': preferred_foot
        }
    END FUNCTION
    
    # Defining the list to store player data
    SET players_list TO EMPTY_LIST()
    SET header_vertical TO EMPTY_SET()  # Use a set to avoid duplicates in the header
    
    SET page TO 1
    WHILE TRUE DO
        SET stats TO fetch_player_stats(page)
        
        IF stats IS EMPTY THEN
            BREAK  # Exit loop if no more data
        
        FOR EACH player IN stats DO
            # Extracting team name from the 'team' field
            SET team_name TO player.get('team', {}).get('name', '') IF player.get('team') IS DICT ELSE ''
            
            SET player_info TO {
                'Team': team_name,  # Updating the 'team' column
                'Player': player.get('player', {}).get('name', '') IF player.get('player') IS DICT ELSE '',
                'PlayerID': player.get('player', {}).get('id', '') IF player.get('player') IS DICT ELSE '',
                # Add all keys and values from player
                **player
            }
            
            # Remove 'player' key to avoid duplication in the DataFrame
            player_info.pop('player', None)
            # Remove 'team' key if it exists since the 'Team' column is created
            player_info.pop('team', None)
            
            APPEND player_info TO players_list
            ADD player.keys() TO header_vertical
        
        INCREMENT page BY 1  # Move to the next page
    
    # Creating DataFrame with player data
    SET df TO CREATE_DATAFRAME(players_list)
    
    # Ensuring all columns are present in the DataFrame, even if absent for some players
    FOR EACH header IN header_vertical DO
        IF header NOT IN df.columns THEN
            SET df[header] TO "N/A"
    END FOR
    
    # Filling additional information for each player
    FOR EACH index, row IN df DO
        SET player_id TO row.get('PlayerID')
        IF player_id EXISTS THEN
            SET details TO fetch_player_details(player_id)
            df.at[index, 'MarketValue'] = details.get('MarketValue', 'N/A')
            df.at[index, 'Age'] = details.get('Age', 'N/A')
            df.at[index, 'Position'] = details.get('Position', 'N/A')
            df.at[index, 'PreferredFoot'] = details.get('PreferredFoot', 'N/A')
        END IF
    END FOR
    
    # Reordering columns to place "PlayerID", "Player", "Team", "MarketValue", "Age", "Position", and "PreferredFoot" first
    SET df TO df[['PlayerID', 'Player', 'Team', 'MarketValue', 'Age', 'Position', 'PreferredFoot', 'appearances', 'rating', 'goals', 'assists'] + [header FOR EACH header IN df.columns IF header NOT IN ['PlayerID', 'Player', 'Team', 'MarketValue', 'Age', 'Position', 'PreferredFoot', 'appearances', 'rating', 'goals', 'assists']]]
    
    # Displaying the DataFrame
    PRINT df
    
    # Exporting to Excel
    EXPORT df TO 'Dataset_Players.xlsx' WITH index FALSE

# Cluster K-Means
    # Load the Excel file
    SET file_path TO 'Dataset_Players.xlsx'
    SET df TO READ_EXCEL(file_path)
    
    # Display the first rows of the dataframe to understand the data structure
    PRINT df
    
    # Select only the numeric columns
    SET numeric_columns TO SELECT_NUMERIC_COLUMNS(df)
    SET df_numeric TO df[numeric_columns]
    
    # Calculate the correlation matrix
    SET correlation_matrix TO CALCULATE_CORRELATION(df_numeric)
    
    # Filter and display only the correlations greater than 0.7 or less than -0.7
    SET filtered_correlation TO correlation_matrix[(correlation_matrix > 0.7) OR (correlation_matrix < -0.7)]
    
    # Remove NaN and zero values to avoid displaying trivial correlations
    SET filtered_correlation TO STACK(filtered_correlation).RESET_INDEX()
    SET filtered_correlation.columns TO ['Variable 1', 'Variable 2', 'Correlation']
    SET filtered_correlation TO filtered_correlation[filtered_correlation['Variable 1'] != filtered_correlation['Variable 2']]
    
    # Sort correlations in ascending order
    SET filtered_correlation TO SORT(filtered_correlation, by='Correlation', ascending=False)
    
    PRINT filtered_correlation
    
    # Save the DataFrame to the Excel sheet
    EXPORT filtered_correlation TO 'correlation_top5.xlsx', sheet_name='Correlation', index FALSE
    
    # Filter players with fewer than 10 games
    SET filtered_df TO df[df['Games'] >= 10]
    
    # Display the filtered data
    PRINT filtered_df
    
    # Save the DataFrame to the Excel sheet
    EXPORT filtered_df TO 'data_top5.xlsx', sheet_name='Data', index FALSE
    
    # Select only the numeric columns
    SET numeric_columns TO SELECT_NUMERIC_COLUMNS(filtered_df)
    
    # Create a DataFrame with only numeric columns
    SET df_numeric_filtered TO filtered_df[numeric_columns]
    
    # Normalize the numeric data
    SET scaler TO StandardScaler()
    SET df_numeric_normalized TO CREATE_DATAFRAME(scaler.fit_transform(df_numeric_filtered), columns=numeric_columns)
    
    # Create a DataFrame with only non-numeric columns
    SET df_non_numeric TO filtered_df.DROP(columns=numeric_columns)
    
    # Reorder to place non-numeric columns at the beginning
    SET df_normalized TO CONCATENATE(df_non_numeric.RESET_INDEX(drop=True), df_numeric_normalized, axis=1)
    
    PRINT df_normalized
    
    # Save the DataFrame to the Excel sheet
    EXPORT df_normalized TO 'datanormalized_top5.xlsx', sheet_name='Data Normalized', index FALSE
    
    # Elbow Method
    
    # List to store inertia values
    SET inertia_values TO EMPTY_LIST()
    
    # Test different numbers of clusters
    FOR num_clusters FROM 2 TO 19 DO
        SET kmeans TO KMeans(n_clusters=num_clusters, random_state=42)
        CALL kmeans.fit(df_numeric_normalized)
        APPEND kmeans.inertia_ TO inertia_values
    END FOR
    
    # Plot inertia graph to identify the "elbow"
    CREATE_FIGURE(figsize=(8, 5))
    CALL plt.plot(range(2, 20), inertia_values, marker='o')
    SET TITLE TO 'Elbow Method'
    SET X_LABEL TO 'Number of Clusters'
    SET Y_LABEL TO 'Inertia'
    CALL plt.grid(True)
    CALL plt.show()
    
    # Silhouette Scores
    
    SET silhouette_scores TO EMPTY_LIST()
    
    # Test different numbers of clusters
    FOR num_clusters FROM 2 TO 19 DO
        SET kmeans TO KMeans(n_clusters=num_clusters, random_state=42)
        SET cluster_labels TO kmeans.fit_predict(df_numeric_normalized)
        
        # Calculate silhouette coefficient score
        SET silhouette_avg TO silhouette_score(df_numeric_normalized, cluster_labels)
        APPEND silhouette_avg TO silhouette_scores
    END FOR
    
    # Plot silhouette coefficient graph
    CREATE_FIGURE(figsize=(8, 5))
    CALL plt.plot(range(2, 20), silhouette_scores, marker='o')
    SET TITLE TO 'Silhouette Coefficient'
    SET X_LABEL TO 'Number of Clusters'
    SET Y_LABEL TO 'Silhouette Coefficient Score'
    CALL plt.grid(True)
    CALL plt.show()
    
    # Choose the number of clusters
    SET num_clusters TO 10
    
    # Apply K-means
    SET kmeans TO KMeans(n_clusters=num_clusters, random_state=42)
    SET df_normalized['Cluster'] TO kmeans.fit_predict(df_numeric_normalized) + 1
    
    # Display the DataFrame with clusters
    PRINT df_numeric_normalized.head()
    
    # Save the DataFrame with clusters to an Excel file
    EXPORT df_normalized TO 'data_top5_clustered.xlsx', sheet_name='Data with Clusters', index FALSE
    
    # Importance of Average Rating
    
    # Separate independent (X) and dependent (y) variables
    SET X TO df_numeric_filtered.DROP(columns=['Average Rating'])
    SET y TO df_numeric_filtered['Average Rating']
    
    # Fit the Random Forest model
    SET rf_model TO RandomForestRegressor(random_state=42)
    CALL rf_model.fit(X, y)
    
    # Get the importance of variables
    SET importances TO pd.Series(rf_model.feature_importances_, index=X.columns).SORT_VALUES(ascending=False)
    
    # Create a DataFrame to save to Excel
    SET importances_df TO importances.RESET_INDEX()
    SET importances_df.columns TO ['Variable', 'Importance']
    
    # Display the importances
    PRINT importances
    
    # Save the DataFrame with Gini to an Excel file
    EXPORT importances TO 'average_rating_top5_gini.xlsx', sheet_name='AverageRating', index TRUE
    
    # Importance of Market Value
    
    # Separate independent (X) and dependent (y) variables
    SET X TO df_numeric_filtered.DROP(columns=['Market Value'])
    SET y TO df_numeric_filtered['Market Value']
    
    # Fit the Random Forest model
    SET rf_model TO RandomForestRegressor(random_state=42)
    CALL rf_model.fit(X, y)
    
    # Get the importance of variables
    SET importances TO pd.Series(rf_model.feature_importances_, index=X.columns).SORT_VALUES(ascending=False)
    
    # Create a DataFrame to save to Excel
    SET importances_df TO importances.RESET_INDEX()
    SET importances_df.columns TO ['Variable', 'Importance']
    
    # Display the importances
    PRINT importances
    
    # Save the DataFrame with Gini to an Excel file
    EXPORT importances TO 'market_value_top5_gini.xlsx', sheet_name='MarketValue', index TRUE
