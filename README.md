# Origin Map Template
A generator to be used for developing new Origin Maps for Factory+. Once the appropriate tag data has been filled in, the sheet `OriginMapInteractive` should be exported as a separate `.csv` file to create the Origin Map.

## Header Definition

The following defines how each field (column in the CSV file) should be filled out for each tag:

### FriendlyName
- **Description**: Name of the tag as it appears on the device. This is used as a human-friendly readable identifier only and is not actually required as part of the translation. If a hierarchical definition is required, it is recommended to use a dot (`.`) separator. This is not mandatory but it is convenient for nested REST calls as it will be in a similar form to the `TagPath` column. There is no restriction on the capitalisation and delimitation characters for multiple word variables within this column but should be consistent across each single Origin Map.
- **Data type**: String
- **Examples**: `VendorName`, `Cell.EmergencyStop`, `Robot.Status`, `map.one_way_map`, `mission.group_id`
- **Optional**: No
### DataType
- **Description**: Data type of tag with indication of endianness. Data types include (see [Sparkplug specification](https://www.eclipse.org/tahu/spec/Sparkplug%20Topic%20Namespace%20and%20State%20ManagementV2.2-with%20appendix%20B%20format%20-%20Eclipse.pdf) for more information):
	- Numeric: Integers (signed and unsigned) `Int8`, `Int16`, `Int32`, `Int64`, `UInt8`, `UInt16`, `UInt32`, `UInt64`. Decimal `Float`, `Double`. Endianness should be included as suffix `BE` - big-endian or `LE` - little-endian (if unsure, the default is usually big-endian)
	- `Boolean` (`true` or `false`)
	- `String`
	- `DateTime` (`UInt64` value representing milliseconds since epoch (Jan 1, 1970))
- **Data type**: String
- **Examples**: `FloatBE`,`UInt16BE`, `Boolean`
- **Optional**: No
### Method
- **Description**: Tag interaction method(s).
	- For **REST**: Use `GET`, `PUT`, `POST`, `DELETE` methods as required. Multiple methods for a single tag can be indicated using comma separated combinations.
	- Use REST style syntax for other protocols too: `GET` for read-only outputs and static values and `PUT` for inputs or write operations.
- **Data type**: String
- **Examples**: `GET`,`PUT`, `GET,POST`
- **Optional**: No


### TagAddress
- **Description**: This `TagAddress` field together with the `TagPath` field uniquely identifies how to access a specific tag from a device. The structure of this field (and of the `TagPath` field) is determined by the protocol being used:
	- **Fieldbus**: Database memory address given as `%aa####.*` where `aa` is replaced by letters which determine the type of variable (eg. boolean input `I` and output `Q`, byte `IB` and `QB`, word `IW` and `QW`, and double word `ID` and `QD`). `####` is replaced by digits representng the byte number of the tag and `*` by a digit representing the bit number.
	- **REST**: Path following the domain in a REST call URL. If using REST to access data from MiR AGVs, the data will be returned in JSON format.

	  One complication here is that recursive function calls may be required to obtain the required data. For example, to obtain whether a robot is "active" or not, you need to first run the `GET /robots` command to return a list of robots together with URLs that will provide more data for each robot. Then, using the URLs, another command of the form `GET /robots/robot_id` needs to be requested and this will return a more detailed JSON response where one of the tags will be the "active" status. So the active status of a robot first requires the output from another request for the list of robot URLs. In such cases, the notation to use is of the form `<PathToList,IdentifiersJsonPath>`, where `PathToList` should be replaced by the REST endpoint to the list of items (in our example `/robots`) and `IdentifiersJsonPath` should use [JSONPath](https://support.smartbear.com/alertsite/docs/monitors/api/endpoint/jsonpath.html) notation (for more information see the [TagPath Section](https://github.com/AMRC-FactoryPlus/origin-map-template#tagpath) below) for a list of all identifiers (`$..url`). The translation app will then cache the data available from each robot's URL in the list of robots and return the available variables for each (one of which is the "active" status ).
	- **MTConnect**: This also uses REST as the protocol but will return a response in the form of XML rather than JSON. Again we must specify the path following the domain in a REST call URL and most of the live data for MTConnect devices can be accessed using `/current`.
	- **OpenProtocol (SmartTools)**: The OpenProtocol MID number and the revision of the MID in the form ```mid****rev###``` where ```****``` is replaced by the 4 digits representing the MID of the message and ```###``` is replaced by the 3 digits representing the revision of the MID.
	- **Other**: TBD
- **Data type**: String
- **Examples**:
	- **Fieldbus**: `%QD1026` or `%Q1194.0`
	- **REST**:`/status` or for recursive calls `</robots,$..url>`
	- **MTConnect**: `/current`
	- **OpenProtocol (SmartTools)**: `mid0040rev001`
	- **Other**: TBD
- **Optional**: No, except for static values which need to be defined in the `Value` column.

### TagPath
- **Description**: This `TagPath` field together with the `TagAddress` field uniquely identifies how to access a specific tag from a device. The structure of this field (and of the `TagAddress` field) is determined by the protocol being used:
	- **Fieldbus** - Leave blank (all of the required information will be provided in the `TagAddress` column).
	- **REST** - If the returned value is in JSON format, [JSONPath](https://support.smartbear.com/alertsite/docs/monitors/api/endpoint/jsonpath.html) notation should be used to uniquely identify the tag. The JSONPath starts with a `$` and each child element is separated by a `.`.
	- **MTConnect**: MTConnect returns data in XML format and there is a standard notation to query XML data called [XPath](https://blog.scrapinghub.com/2016/10/27/an-introduction-to-xpath-with-examples) (analogous to JSONPath). The notation recommended here is to use the select all operator `//*` and then identify a specific tag by searching for a specific attribute name `[@attributeName="test"]`. We can uniquely identify data in MTConnect responses by using the attributes `componentId` and `dataItemId`. To obtain the text content (actual data values) of an XML node, we can use `/text()` at the end of the attribute query.

	  In some cases (such as MTConnect Conditions) it may be required to obtain the name of the XML node (eg. Normal, Warning or Fault for an MTConnect Condition) itself rather than just the text associated with the node. Here, we must use `local-name()` and place the attribute query within the parenthesis. Note: appending `/local-name()` to the end of the attribute query will not work.
	- **OpenProtocol (SmartTools)**: Even though the returned response is not in JSON format, for consistency, we are using the `$.` notation provided by [JSONPath](https://support.smartbear.com/alertsite/docs/monitors/api/endpoint/jsonpath.html) to define which particular parameter to extract from a message.
	- **Other**: TBD
- **Data type**: String
- **Examples**:
	- **Fieldbus**: leave empty
	- **REST**: `$.status.errors.code` or to return a tag from the first value in an array `$[0].state`
	- **MTConnect**: x-axis position: `//*[@componentId='x']//*[@dataItemId='xp']/text()`, hydraulic condition: `local-name(//*[@componentId='hydraulic']//*[@dataItemId='hyd'])`, message associated with the hydraulic condition:`//*[@componentId='hydraulic']//*[@dataItemId='hyd']/text()`
	- **OpenProtocol (SmartTools)**: `$.cellID`, `$.toolMaxTorque` and `$.disableTool`
	- **Other**: TBD
- **Optional**: No, except for static values which need to be defined in the `Value` column.



### Value
- **Description**: Can be used to manually enter the value of a specific tag:
	- For inputs and output tags - default or startup value.
	- For static tags - value definition.
- **Data type**: As specified in `DataType` column
- **Examples**: Setting a numerical value `100`, or a boolean value `FALSE`, or a static string such as manually entering a robot serial number `RoBoT5678`.
- **Optional**: Yes, except for static values with empty `TagAddress` or `TagPath` columns.

### EngUnit
- **Description**: Unit associated with the tag. The notation used for the unit must be as specified by the DisplayName of the OPC UA Engineering units (this is an extensive [list of available units](http://www.opcfoundation.org/UA/EngineeringUnits/UNECE/UNECE_to_OPCUA.csv), and following the OPC notation will prevent duplication or ambiguity of units). Can leave this empty to signify that a unit is not applicable to the tag.
- **Data type**: String
- **Examples**: `mm`, `%`, `°`, `m/s`, `°C`
- **Optional**: Yes
### EngLow
- **Description**: Tag minimum possible value, for example, the hardware limit.
- **Data type**: As specified in `DataType` column
- **Examples**: `0`, `-100`
- **Optional**: Yes
### EngHigh
- **Description**: Tag maximum possible value, for example, the hardware limit.
- **Data type**: As specified in `DataType` column
- **Examples**: `1`, `100`
- **Optional**: Yes
### Deadband
- **Description**: Numerical value used to prevent unnecessary updates for tags whose values _float_ by small amounts. If a tag value has changed by less than the deadband amount, the tag will not be updated to the new value. Particularly useful for tags that frequently fluctuate by a small amount.
- **Data type**:
	- Float - For specifying absolute value change.
	- Float with `%` symbol - For specifying percentage change based on `EngLow` and `EngHigh`. `EngLow` and `EngHigh` values are required for this mode.
- **Examples**: `3.75`, `1.5%`
- **Optional**: Yes

### RecordToHistorian
- **Description**: Indicates if tag values will be recorded to the Factory+ Historian Database and should  usually be set to true.
- **Data type**: Boolean
- **Examples**: `TRUE` or `FALSE`
- **Optional**: No, `TRUE` by default.

### DeviceType
- **Description**: The `DeviceType`, `DeviceName` and `CDSName` columns together form the entire Common Data Structure (CDS) path to where a specific tag will be stored on the Factory+ network. This `DeviceType` column is the top-level folder in the CDS and indicates the category or type of device. There are a limited number of device types provided with the existing CDSs and each `DeviceType` dictates the folder structure (the contents of the `DeviceName` and `CDSName` columns) for the tags to be stored.

  For any tags that do not belong under an existing `DeviceType`, the user can create a custom structure using the `User_Defined` device type. The naming convention for all folders and variable within `User_Defined` should be `Pascal_Snake_Case` if possible or be inherited from the device/protocol in use.
- **Data type**: String
- **Examples**: `MotionDevices`, `CncAxisList`, `Smart Tools`, `User_Defined`
- **Optional**: No

### DeviceName
- **Description**: The `DeviceType`, `DeviceName` and `CDSName` columns together form the entire CDS path to where a specific tag will be stored on the Factory+ network. While the `DeviceType` column provides the top-level category in the CDS, this `DeviceName` column is the second level in the CDS (child folder of `DeviceType`) and provides a specific name to the device. This can be a user created name for the device to distinguish between other devices stored under the same `DeviceType` or if possible, extracted as a tag (such as the robot name) from the device itself.

  Note that when using custom `User_Defined` variables, some kind of `DeviceName` should always be provided to give context as to which device the tag relates to. This can be a `DeviceName` that has already been used in the Origin Map (if the tag is related to other tags in the Origin Map, but there is no default CDS path provided for this type of tag) or can be an arbitrary new name for tags that don't fit under any existing `DeviceName`. Any new folder structures, should be created with clear, concise and logical groups and hierarchies of tags so that if required and following consultations with the Factory+ working group, could lead to the creation of new CDSs or new sections in existing CDSs.
- **Data type**: String
- **Examples**: `Robot1`, `Y Axis`, `Bosch Nexo Nutrunner 1`
- **Optional**: No

### CDSName
- **Description**: The `DeviceType`, `DeviceName` and `CDSName` columns together form the entire CDS path to where a specific tag will be stored on the Factory+ network. While the `DeviceType` column provides the top-level category in the CDS, and the `DeviceName` column provides the second level name of the device in the CDS (child folder of `DeviceType`), this `CDSName` column provides the rest of the folder structure to reach the tag. Each level of folder should be delimited by a slash `/`. Additionally, some tags (within angle brackets `<`, `>`) are placeholders and should be replaced with appropriate names/identifiers. They usually occur one level after a category/array folder and are used to identify elements within the category. Example `CDSName` include `Axes/<Axis Name>/ActualPosition` (from Robot CDS) or `Mission_List/<Mission_Name>` (from AGV CDS) and the contents in the angle brackets would be replaced by an actual identifier for the axis or an actual name of the mission. The name/identifier can be manually entered as an arbitrary string by the user (such as `Joint1` for the axis) or be extracted as tag data from the device itself (for the JSON messages of the MiR AGVs, can provide data using [JSONPath](https://support.smartbear.com/alertsite/docs/monitors/api/endpoint/jsonpath.html) notation within the angle brakets, as well as providing the appropriate `TagAddress` and `TagPath` columns in the Origin Map to extract the name of the AGV mission).
- **Data type**: String with forward slash (`/`) category separation.
- **Examples**: `Axes/Joint1/ActualPosition`, `AnglePos/ActPos`, `Robot_List/5/Current_Position/Pos_X`, or using JSONPath notation to extract all the mission names `Mission_List/<$..name>`
- **Optional**: No


### Tooltip
- **Description**: Short tag description that will display as a Tooltip when hovering over the tag. Can be simple text or could try and include more context such as enum value definitions.
- **Data type**: String
- **Examples**: `Spindle Speed`
- **Optional**: Yes
### Documentation
- **Description**: More detailed tag description, possibly including links to manuals. This can be copied from the `description` property of the tag as provided by the JSON file of a CDS. Or it can alternatively be written from scratch and should to try to provide enough context so that an uninvolved reader could understand what is being stored for this tag.
- **Data type**: String
- **Examples**: `The current name of the fleet`
- **Optional**: Yes
