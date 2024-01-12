# Config custom snippets

1. Ensure language is enable quick suggestion
   open settings.json add below code
   ```json
   "[%LANGUAGE%]": {
        "editor.quickSuggestions":true
        },
   ```
   

2. 「ctrl + shift + ;」 open language selector
3. search 「code snippets」
4. edit json as comment

# Add auto-close
1. Open json file <font color='#EEB422'>「%VS Code install path% resources\app\extensions\markdown-basics\language-configuration.json」</font>
2. add new item in「autoClosingPairs」object 