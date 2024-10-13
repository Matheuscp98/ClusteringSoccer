# Clustering Soccer

    ## Extract Data: League - Year
    
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
    EXPORT df TO 'player_statistics_22_br.xlsx' WITH index FALSE
