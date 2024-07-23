```js
(async () => {
    while (true) {
        // zoho recruit token
        const token = '1000.e4d32a2748a87b09aa89eac779754fed.8b414b8bc55f63244b539cbc6576deda';
        const url = 'https://recruit.zoho.eu/recruit/v2/settings/fields?module=Candidates';

        let res;
        try {
            res = await fetch(url, { headers: { 'Authorization': `Zoho-oauthtoken ${token}` } });
        } catch (error) {
            return console.error(error);
        }

        const headers = Object.fromEntries(Array.from(res.headers));
        const headerKeys = Object.keys(headers);
        const zohoRecruiteLimitKey = headerKeys.find(key => key.toLowerCase() === 'x-ratelimit-remaining');

        if (!zohoRecruiteLimitKey) {
            console.log('исчерпали дневной лимит');
            return;
        }
        
        if (Number(headers[zohoRecruiteLimitKey]) === 0) {
            console.log('исчерпали минутный лимит');
            await new Promise(res => setTimeout(res, headers['x-ratelimit-reset'] - Date.now()));
            console.log('минутный лимит восстановлен, продолжаем запросы');
        }
    }
})();
```
