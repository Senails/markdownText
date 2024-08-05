```js
(async () => {
    while (true) {
        // capsule crm token
        const token = 'zwnPOSneFXUeyqSLgEFdaLhXtL1Fwp7noZu3GJHL33EQ5GUSKtO7mfTgXV0Ndq30';
        const url = 'https://api.capsulecrm.com/api/v2/parties/fields/definitions?perPage=100&page=1';

        let res;
        try {
            res = await fetch(url, { headers: { 'Authorization': `Bearer ${token}` } });
        } catch (error) {
            return console.error(error);
        }

        const headers = Object.fromEntries(Array.from(res.headers));
        const limit = Number(headers['x-ratelimit-limit']);
        const currentLimit = Number(headers['x-ratelimit-remaining']);

        if (currentLimit > 0) {
            console.log(currentLimit + '/' + limit)
            // break;
        } else {
            console.log(currentLimit + '/' + limit)
            break;
        }
    }
})();
```
