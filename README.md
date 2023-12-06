```js
const FETCH_CONFIG = {
    headers:{
        "Tenant-Organization2": "5c916c11-d2f0-4c6b-aa34-be3b41942057",
        "Tenant-Workspace2": "5c9176af-ad29-45f1-94f0-7e5e1d9491c0",
        "Content-Type": "application/json"
    }
};
const CUSTOM_FIELD_NAME = 'Actual dev';
const LOCAL_STORAGE_KEY = 'ActualDevFixStoriId';
const LOCAL_STORAGE_ERROR_KEY = 'ActualDevErrorMessage';
const customFieldsJson = await (await fetch('https://app.shortcut.com/backend/api/private/custom-fields',FETCH_CONFIG)).json();
const ACTUAL_DEV_FIELD_ID = customFieldsJson.find((elem) => elem.name === CUSTOM_FIELD_NAME).id;

async function getHistoryByStoryID(id){
    let res = await fetch(`https://app.shortcut.com/backend/api/private/stories/${id}/history`, FETCH_CONFIG);
    let json = await res.json();
    return json;
}

const CUSTOM_FIELD_NAMES = ['Actual dev','Actual'];

// 0.1 Проверка NAME
// применяется только в получении данных из истории 
function checkName(name) {
    return CUSTOM_FIELD_NAMES.includes(name);
}

// 1 функция для получения числа если оно есть в истории
async function findActualDevByHistory(id) {
    const historyArr = await getHistoryByStoryID(id);
    const elem = historyArr.findLast((el)=>{
        if (!el.references) {
            return false;
        }
        const reference = el.references.find( ref => checkName(ref.field_name));
        return !!reference;
    });
    if (!elem) {
        return;
    }
    const reference = elem.references
        .find( obj => obj.field_name && checkName(obj.field_name));
    if (!reference) {
        return;
    }
    const text = reference.string_value;
    return parseInt(text);
}
// 2 функция для получения value_id по числу
const getActualDevValueIdByCount = await (async () => {
    const actualDev = customFieldsJson
        .find((elem) => elem.name === CUSTOM_FIELD_NAME);
    const actualDevValues = actualDev.values;
    return (num) => {
        const countIdObj = actualDevValues
            .find((elem)=> parseInt(elem.value) === num );
        return countIdObj.id;
    }
})();
// 3 функция для получения текущего числа актуал дев у стори
function getActualDevByStory(story){
    const actualDevFieldObj = story.custom_fields
        .find( elem => elem.field_id === ACTUAL_DEV_FIELD_ID );
    if (!actualDevFieldObj) {
        return undefined;
    }
    return parseInt(actualDevFieldObj.value);
}
// 3.1 функция для получения storyObject по StoryId
async function getStoryObjectByStoryId(id) {
    const res = await fetch(`https://app.shortcut.com/backend/api/private/stories/${id}`, FETCH_CONFIG );
    const storyObject = await res.json();
    return storyObject;
}
// 4 Функция для внесения изменений в актуал дев стори
async function setActualDevValue(story, count){
    let outputCustomFields = story.custom_fields.map(({field_id, value_id}) => {
        return {
            field_id,
            value_id,
        };
    });
    let objForChange = outputCustomFields.find(({field_id}) => field_id === ACTUAL_DEV_FIELD_ID);
    if (objForChange) {
        objForChange.value_id = getActualDevValueIdByCount(count);
    } else {
        outputCustomFields.push({
            field_id:  ACTUAL_DEV_FIELD_ID,
            value_id: getActualDevValueIdByCount(count),
        })
    }
    await fetch(`https://app.shortcut.com/backend/api/private/stories/${story.id}`,{
        method: 'PUT',
        headers: FETCH_CONFIG.headers,
        body:JSON.stringify({
            custom_fields: [...outputCustomFields]
        }),
    });
}
// 5 обобщенная функция для того что бы починить сторю
// получает сторю , ищет в ней актуал дев
// если актуал дев есть то завершается
// если актуал дев пуст то пробует узнать актуал дев с помощью истории
// если не нашел в истории кидает ошибку
// если нашел меняет текущий актуал дев
async function fixActualDevValue(storyId){
    const story = await getStoryObjectByStoryId(storyId);
    if (story.message && story.message === 'Resource not found.') {
        console.log(`Стори #${storyId}: не найдена`);
        return;
    }
    const actualDev = await getActualDevByStory(story);
    if (actualDev) {
        console.log(`Стори #${storyId}: уже имеет ActualDev (${actualDev} point${actualDev === 1 ? '' : 's' })`);
        return;
    }
    const actualDevByHistory = await findActualDevByHistory(storyId);
    if (!actualDevByHistory) {
        console.log(`Стори #${storyId}: ActualDev не найден в истории`);
        return;
    }
    // await setActualDevValue(story, actualDevByHistory);
    console.log(`Стори #${storyId}: был успешно задан ActualDev (${actualDevByHistory} point${actualDevByHistory === 1 ? '' : 's' })`);
    return;
}
// 6 processStory
// функция должна бежать по сторям и чинить их
async function processStories(rangeStart = 1, rangeEnd){
    if (!rangeEnd) {
        throw new Error('задавать конечное значение обязательно!');
    }
    let storyId = rangeStart;
    try {
        while( storyId < rangeEnd ){
            await fixActualDevValue(storyId);
            localStorage.setItem(LOCAL_STORAGE_KEY,`${storyId}`);
            storyId++;
        }
    } catch (error) {
        let e = {
            storyId,
            message: error.message,
        };
        localStorage.setItem(LOCAL_STORAGE_ERROR_KEY, JSON.stringify(e));
        console.error(e);
    }
}


await processStories(1, 17500);
```
