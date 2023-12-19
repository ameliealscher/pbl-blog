---
layout: post
title: "searching for books inside the Google Books API"
---
## My problem
A requirement for my application was to be able to search books by titles or authors in the Google Books API. After entering a keyword into the searchbar and clicking search, a list of books should appear. These books should also immediately be saved to the database.

The challenge involves integrating the Google Books API into an application, searching for books by title or author, retrieving a list of books, and saving them to a database. Initially, the implementation used manual HTTP requests and JSON parsing, but it was not considered optimal. To improve the approach, a WebClient and Jackson library were employed for a more efficient solution.

### Solution
I put a lot of work into researching the implementation. Initially, I solved this problem by using manual HTTP requests and JSON parsing. Later on I came to the conclusion, that this, even if it works, is not optimal. So I decided to spend even more time researching.

This was my initial implementation of the HTTP Request inside the searchBooks function:
```
            URL url = new URL(apiUrl);
            HttpURLConnection connection = (HttpURLConnection) url.openConnection();
            connection.setRequestMethod("GET");

            BufferedReader reader = new BufferedReader(new InputStreamReader(connection.getInputStream()));
            StringBuilder response = new StringBuilder();
            String line;

            while ((line = reader.readLine()) != null) {
                response.append(line);
            }
            reader.close();
            connection.disconnect();
```
This was my initial implementation of the JSON parsing inside the searchBooks function:
```
JSONObject jsonResponse = new JSONObject(response.toString());
            JSONArray items = jsonResponse.getJSONArray("items");

            for (int i = 0; i < items.length(); i++) {
                JSONObject volumeInfo = items.getJSONObject(i).getJSONObject("volumeInfo");
                String title = volumeInfo.getString("title");

                JSONArray authorsArray = volumeInfo.optJSONArray("authors");
                List<String> authorList = new ArrayList<>();

                if (authorsArray != null) {
                    for (int j = 0; j < authorsArray.length(); j++) {
                        authorList.add(authorsArray.getString(j));
                    }
                } else {
                    String author = volumeInfo.optString("authors", "N/A");
                    authorList.add(author);
                }
                books.add(new Book(title, authorList, List.of()));
            }
```
After spending even more time researching, I decided on making the HTTP request using WebClient, this simplifies the process of making HTTP requests in a reactive way.

This is my new implementation of the HTTP Request inside the searchBooks function:
```
 JsonNode jsonResponse = webClientBuilder.build()
                    .get()
                    .uri(apiUrl)
                    .retrieve()
                    .bodyToMono(JsonNode.class)
                    .block();
```
I also decided on parsing the JSON manually, because of the way the Google Books API items are built. But I did that with the help of Jackson, because it's built into Spring Boot and it makes my code more readable and easier to maintain.

This is my new implementation of the JSON parsing inside the searchBooks function:
```
 if (jsonResponse != null) {
                JsonNode items = jsonResponse.get("items");

                if (items != null && items.isArray()) {
                    for (JsonNode item : items) {
                        JsonNode volumeInfo = item.get("volumeInfo");
                        String title = volumeInfo.get("title").asText();

                        JsonNode authorsArray = volumeInfo.get("authors");
                        List<String> authorList = new ArrayList<>();

                        if (authorsArray != null && authorsArray.isArray()) {
                            for (JsonNode authorNode : authorsArray) {
                                authorList.add(authorNode.asText());
                            }
                        } else {
                            String author = volumeInfo.path("authors").asText("N/A");
                            authorList.add(author);
                        }
                        books.add(new Book(title, authorList, List.of()));
                    }
                }
            }
```