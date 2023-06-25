# Integration - Maps:

## Project Details
- This project is a full stack web application that allows users and developers to exercise 3 main uses/functionalities:
    - Users can view a map of a particular area that they’re interested in at any zoom level
    - Users can see an overlay of historical redlining data atop a map that they’re viewing as well as be shown the state, city, and name of the area if they are defined in the data
    - Developers can use the API server contained in the web app to not only access the redlining `GeoJSON` dataset but also filter the data by providing a bounding box query

# Design Choices:

## Packaging and Relationship Between Classes/Interfaces:

### Frontend:
- To start, I separated the code into 2 main packages: `frontend` and `backend`. The `frontend` folder contains all of the React code and the files necessary for creating the Map component and the user interface. 
- The `backend` folder contains all of the server code (with extended functionality for the map endpoint handler) that creates and runs the API server.
- Within the `frontend` folder, the sub-packages and files follow from the structure of the default React application, with the `public` folder containing many of the necessary configuration files and the `src` folder containing each of the command line UI components.
- The file `App.tsx` contains the code for the main React component that is rendered to the DOM by ReactDOM. It has an app header and also contains the `MapComponent` component within the JSX.
- Within `src`, the `MapComponent` file is used to create an interactive map in the project, which is done by making an API call to `Mapbox` using all the properties I created the Map component with.
- I then create a file called `Overlay.ts` which contains a file that allows us to overlay data on top of the map. I do this by creating a `FillLayer` object that I export to `MapComponent.tsx` in order to use for the `Layer` component that is a subcomponent of the Map JSX. Within `Overlay.ts`, there are also functions to fetch the GeoData json from the backend to overlay on the map. 

### Changes Made to the Frontend:
- To begin, I created 3 new state variables that represented the state, city, and name properties that I wanted to display to the user. I also implemented a `clickStatus` variable to indicate whether or not a certain portion of the map has been clicked on already.
- From there, I retrieve the selected features of the map by using the `queryRenderedFeatures()` method provided by `Mapbox` and go through each of the properties associated with that map click. The `onMapClick()` function then returns a list containing the state, city, and name of that particular location for us to display.
- Lastly, in the `Overlay.ts` file, I added in a function that provided the application with a way to fetch the `GeoJSON` redlining data directly from the backend without having to import it within a particular frontend file.

### Backend:
- Within the `backend` folder, I organized the files and sub-folders using the following structure (similar to Sprint 2):
    - A `csv` folder that handled parsing and includes the word-count utility (from Sprint 0)
    - A `server` folder that contains the subfolder `api` (which contains the utilities and handlers sub-packages) and weather classes (used for processing the NWS API response)
        - The `utilities` folder contains `weatherUtilities`, which provides methods for de-serializing and serializing between the Weather API’s JSON responses and nested Java Objects capturing the JSON’s structure.
        - The handlers folder contains handle methods for the API’s endpoints, “/getcsv”, “/loadcsv”, and “/weather”. The last handle method `UnsupportedResponseHandler` is for any other end point that is currently not a valid get request for the web API.
            - I added an additional handler to this package (`StatsCSVHandler`) to handle the scenario in which the user enters a "stats" command into the command line interface.
        - `weather.classes` contain Java classes that nest within each other, mirroring the nested fields in the weather API’s response.
        - The server folder also contains the `DataSource` Class representing the shared state variable, and `Server` (which, when run, creates the web API).
- The rest of the folders are packaged the way they were in Sprint 0 (`stars`, `main`, `kdtree`).

### Changes Made to the Backend:
- In the `server` class, I added an additional endpoint called “map.” I also created a new API handler called `MapHandler` which implements `Route` so that it is compatible with Spark.
- In the `handle` method within `MapHandler`, I check the parameters of the bounding box (i.e. `minLat`, `minLon`, `maxLat`, `maxLon`) to make sure that they are present in the query. 
    - If there are any invalid or missing parameters, I add an error message to the map contained in the `Result` object and then return the serialized `Result`. 
    - Afterwards, I create a new `Result` object which is the object that should be serialized through the `Serializer.java` class.
    - Subsequently, I first convert all of the query parameters into doubles so that they are easier to compare (in comparison to `Strings`). After that, I create a new `MapDeserialize` object and convert the `GeoJSON` data into a string using the `readString()` method.
    - From there, I deserialize this string into a custom `GeoData` object to represent the unfiltered data. This is done in the `MapDeserializer` class using `Moshi` and `JsonAdapter`.
    - I then pass this unfiltered data along with the bounding box into the `filterJson` method, which goes through the data and adds each relevant feature to a list of `Feature` objects that I then return at the end of the program.
- Within the `server` folder, I also created 4 separate classes in order to deserialize the nested `GeoJSON` into its properties. These classes are `Feature`, `GeoData`, `Geometry`, and `Property`.
    - Each `Feature` object contains a String `type`, a geometry field of type `Geometry`, and a hashmap properties that maps a String to an Object.
    - Each `GeoData` object contains a string representing its type as well as a list of `Feature` objects features.
    - I created the `Geometry` class so that it could more accurately model the coordinates stored within the `GeoJSON`. I did this by continuing to keep a String type but I also added a `List<List<List<double[]>>>` in order to mimic the structure of the coordinate pair data.
    - Lastly, the `Properties` class simply contains strings representing the `state`, `city`, `name`, `holc_id`, `holc_grade`, `neighborhood_id`, and a hashmap that maps a string to another string representing the area description data.

### Data Structures:
The most notable data structure that I used was to create a `GeoData` object in order to store the deserialized version and properties of the `GeoJSON` data.
Additionally, I decide to include fields for `Feature` and `Properties` using a hashmap to allow for easy retrieval of certain data associated with a particular key value.

### Runtime/Optimizations:
- I made no significant runtime optimizations. However, I did include a `useEffect` hook within the `MapComponent.tsx` file in order to prevent the page from re-rendering every time the user clicks on an area of the map. I figured that making a request to the API every time the view state changes would be inefficient.
- Another area that I were considering optimizing for was when I were looping through each of the items in the `GeoJSON` data. In most occasions, iterating through everything pair of coordinates in the JSON could be very computationally expensive; however, I decided that in order to be thorough with the filtering, this was a necessary runtime compromise that I needed to make.

## Tests for Integration: Maps:

### Unit Testing for the Bounding-Box Filtering Requirement on Map Data:
- I did this testing in `testBoundingBox.java` in the backend
- In order to verify the functionality of the bounding-box filtering requirement, I essentially unit tested the `filterJson()` method in the backend. To do this, I create mock `GeoData` objects that I pass into the method, along with the 4 necessary parameters (`minLon`, `maxLon`, `minLat`, `maxLat`) to see if I receive the expected features within the `GeoData` object I return. I tested for the following cases:
    - No `Feature` objects are added to the list because bounding box dimensions are the same
    - 1 `Feature` object is added in the final list
    - 2 `Feature` objects are added in the final list
    - All 3 `Feature` objects from the original mock list are added in the final list
    - All query parameters are 0
    - Testing using a `GeoData` object with an empty list of `Feature` objects
- I also added several integration tests to verify the backend functionality of the web application. I included the boilerplate setup and teardown methods at the beginning of the test suite. In each test, I create a mock `HttpURLConnection` object in order to simulate a call to the backend API server with the map request. 
- I then use the `Moshi` and `JsonAdapter` features to check that the user’s request is correctly serialized to the feature included in each `Result` object’s map field. I test for the following cases
    - Successful endpoint request with the map parameter
    - Query parameters have number formatting issues
    - No parameters for the map request are entered
    - Parameters for the map request are missing
    - Testing a basic connection to the backend API server

### Mock Infrastructure Test:
In `TestMockMap.java` in the backend, I created a mock `GeoJSON` with varying coordinates and missing fields from the default GeoJSON structure delineated in the redlining JSON I were given to make sure that I could support this new format and deserialize it as well.

### Fuzz Testing/Random Testing:
- I created a testing file called `TestFuzzTesting.java` in the backend that has a method to generate four random parameters (`minLon`, `maxLon`, `minLat`, `maxLat`). 
- I then call this method 1,000 times to generate the random parameters and then I pass it into the `filterJson()` method.

## Other Notable Design Choices:
**Defensive Programming:** I made the Moshi instance variable private and final to prevent any modifications to this variable.
**Accessibility:** I added `aria-labels` on the frontend in order to make the state, city, and name of a particular location get read out to the user when they click on a portion of the map.

### Running the Tests:
- On Windows, search “Command Prompt” in the menu in the bottom left corner of the screen
- Click the menu and run the program when it appears.
- On Mac, search “terminal” to open the application, navigate to the `backend` folder, and then run `mvn site` to run all test suites.

### Build and Run the Program:
- In order to start the web application, I have to run the backend `server` class.
- Afterwards, I navigate to the frontend React app, navigate to the `Map` folder, and run `npm start` (run `npm install` if `npm` is not already downloaded)
