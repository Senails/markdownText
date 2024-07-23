```js
(async () => {
    while (true) {
        // zoho crm token
        const token = '1000.701ec66f1d74f1cff67f6c3d6bcbb763.91e9c15091724b686a448ead7360217c';
        const url = 'https://www.zohoapis.eu/crm/v6/settings/fields?module=Leads';

        let res;
        try {
            res = await fetch(url, { headers: { 'Authorization': `Zoho-oauthtoken ${token}` } });
        } catch (error) {
            return console.error(error);
        }

        const headers = Object.fromEntries(Array.from(res.headers));
        const headerKeys = Object.keys(headers);
    
        const neededHeader = 'X-API-CREDITS-REMAINING';
        const key = headerKeys.find(key => key.toLowerCase() === neededHeader.toLowerCase());

        if (!key) continue;
        
        const apiCreditsCount = headers[key];
        if (apiCreditsCount > 0) continue;
        
        return console.log(apiCreditsCount);
    }
})();
```
