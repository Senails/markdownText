```js
const FETCH_CONFIG = {
    headers:{
        "Tenant-Organization2": "5c916c11-d2f0-4c6b-aa34-be3b41942057",
        "Tenant-Workspace2": "5c9176af-ad29-45f1-94f0-7e5e1d9491c0",
        "Content-Type": "application/json"
    }
};

// const customFieldsJson = await (await fetch('https://app.shortcut.com/backend/api/private/custom-fields',FETCH_CONFIG)).json();
// const membersListJson = await (await fetch('https://app.shortcut.com/backend/api/private/members',FETCH_CONFIG)).json();

async function getHistoryByStoryID(id){
    let res = await fetch(`https://app.shortcut.com/backend/api/private/stories/${id}/history`, FETCH_CONFIG);
    let json = await res.json();
    return json;
}
async function getStoryObjectByStoryId(id) {
    const res = await fetch(`https://app.shortcut.com/backend/api/private/stories/${id}`, FETCH_CONFIG );
    const storyObject = await res.json();
    return storyObject;
}

const excludeEditorsList = [
    'Alla Sitnikova',
];

// функция для получения массива людей списывающих
// определенные кастомные поля из истории
function getSpandings(storyHistory, fieldNames) {
    let actualValue = 0;
    
    const changes = storyHistory
        .filter( obj => obj.references)
        .filter( obj => obj.references
            .find( elem => fieldNames.includes(elem.field_name)))
        .map ( obj => {
            const member = getMemberNameById(obj.member_id);
            let countChanges;
                
            if (obj.references.length === 1) {
                countChanges = parseInt(obj.references[0].string_value) - actualValue;
            } else {
                countChanges = parseInt(obj.references[0].string_value) - parseInt(obj.references[1].string_value);
            }
            
            actualValue += countChanges;

            return {
                member,
                countChanges,
            }
        })
        .filter ( ({member}) => !excludeEditorsList.includes(member));
        
    let dictionary = {};
    changes.forEach( change => {
        if (dictionary[change.member]) {
            return dictionary[change.member] += change.countChanges;
        }
        dictionary[change.member] = change.countChanges;
    })
    
    return Object.entries(dictionary).map( ([member, total_hours]) => ({member, total_hours}))
}

// поиск имени по айди пользователя
function getMemberNameById(memberId) {
    const member = membersListJson
        .find( m => m.id === memberId);

    if (member) {
        return member.profile.name;
    }
    return memberId;
}

// получаем значение поля по его имени
function getCustomFieldValue( story, fieldName) {
    const field = customFieldsJson.find( e => e.name === fieldName);
    if (field) {
        const obj = story.custom_fields.find( e => e.field_id === field.id);
        if (obj) {
            return obj.value;
        }
    }
    return '';
}

// получаем время сумарное время ннахождения 
// в определенном состоянии
function getTotalTimeInState( stateChangesHistory, state) {
    if (!stateChangesHistory.find( e => e.state === state )) {
        return 0;
    }
    
    const summ = stateChangesHistory
        .map( (obj, index, array) => {
            if (obj.state !== state) {
                return 0
            }
            const date1 = new Date(obj.date);
            const date2 = new Date(array[index + 1]?.date || Date.now());        
            return (date2 - date1);
        })
        .reduce((partialSum, e) => partialSum + e, 0);
    
    return Math.floor(summ/(60*60*24))/1000;
}

// поиск разницы в днях между датами
function getDaysBetweenDates( dateIso1 , dateIso2) {
    if (dateIso1) {
        const date1 = new Date(dateIso1);
        const date2 = new Date(dateIso2 || Date.now());

        return Math.floor((date2 - date1)/(60*60*24))/1000;
    }
    return null;
}

// функция для сбора статистики с стори
function getStoryStats(story, storyHistory) {

    const stats = {
        story_id: story.id,
        story_name: story.name,
        
        type: [story.story_type[0].toUpperCase(), 
               ...story.story_type.slice(1)
        ].join(''),

        owner: getMemberNameById(story.owner_ids.find(() => true)),
    };

    stats.actual_dev_spendings = getSpandings(storyHistory, ['Actual', 'Actual dev']); // реализовать функцию
    stats.actual_review_spendings = getSpandings(storyHistory, ['Actual review']);
    stats.actual_qa_spendings = getSpandings(storyHistory, ['Actual QA']);
    
    stats.owners_from_history = stats.actual_dev_spendings.map( ({member}) => member);
    stats.reviewers_from_history = stats.actual_review_spendings.map( ({member}) => member);

    stats.reviewer = getCustomFieldValue( story, 'Reviewer' ) || null;
    stats.qa = getCustomFieldValue( story, 'QA' ) || null;

    stats.actual_qa = parseInt(getCustomFieldValue( story, 'Actual QA' )) || 0;
    stats.actual_dev = parseInt(getCustomFieldValue( story, 'Actual dev' )) || 0;
    stats.actual_review = parseInt(getCustomFieldValue( story, 'Actual review' )) || 0;

    stats.pulls_count = story.pull_requests.length;
    stats.pull_links = story.pull_requests.map( e => e.url );

    stats.estimate_last_value = story.estimate || 0;
    stats.estimate_qa = parseInt(getCustomFieldValue( story, 'Estimate QA' )) || 0;

    let estimateHistory = storyHistory
        .filter( e => e.actions && e.actions.length && e.actions
            .find( action => action.changes && action.changes.estimate))
        .map( change => change.actions
            .find( action => action.changes && action.changes.estimate)
            .changes.estimate.new);
    
    stats.estimate_first_value = estimateHistory[0] || null; 
    stats.estimate_second_value = estimateHistory[1] || null;

    let estimateQaHistory = storyHistory
        .filter( e => e.references && e.references.length && e.references
            .find( ref => ref.field_name === 'Estimate QA'))
        .map( e => e.references
            .find( ref => ref.field_name === 'Estimate QA').string_value)
        .map( str => parseInt(str));
    
    stats.estimate_qa_first_value = estimateQaHistory[0] || null;
    stats.estimate_qa_second_value = estimateQaHistory[1] || null;

    const stateChangesHistory = storyHistory
        .filter( e => e.references && e.references.length && e.references
            .find( ref => ref.entity_type === 'workflow-state'))
        .map( e => {
            return {
                state: e.references
                    .find( ref => ref.entity_type === 'workflow-state').name,
                date: e.changed_at,
            }
        });

    stats.state = stateChangesHistory[stateChangesHistory.length - 1].state;
    
    stats.state_changes_to_in_development = stateChangesHistory
        .filter( obj => obj.state === 'In Development').length;
    
    stats.state_changes_to_in_qa = stateChangesHistory
        .filter( obj => obj.state === 'In QA').length;
    
    stats.state_changes_to_ready_for_review = stateChangesHistory
        .filter( obj => obj.state === 'Ready for Review').length;

    stats.story_created_at = story.created_at;
    stats.story_completed_at = story.completed_at;

    stats.first_move_to_in_development = stateChangesHistory
        .find(obj => obj.state === 'In Development')?.date || null;
    
    stats.last_move_from_in_development = stateChangesHistory
        .find( (obj, index, array) => array[index - 1]?.state === 'In Development')?.date || null;

    stats.days_between_first_move_in_and_last_out_development =
        getDaysBetweenDates( stats.first_move_to_in_development, stats.last_move_from_in_development);
        
    stats.days_between_start_development_and_completed =
        getDaysBetweenDates( stats.first_move_to_in_development, stats.story_completed_at);
    
    stats.days_between_created_and_completed =
        getDaysBetweenDates( stats.story_created_at, stats.story_completed_at);
    
    stats.total_days_in_development = getTotalTimeInState(stateChangesHistory, 'In Development');
    stats.total_days_ready_for_review = getTotalTimeInState(stateChangesHistory, 'Ready for Review');
    stats.total_days_ready_for_qa = getTotalTimeInState(stateChangesHistory, 'Ready for QA');
    stats.total_days_in_qa = getTotalTimeInState(stateChangesHistory, 'In QA');
    
    return stats;
}

// const story = await getStoryObjectByStoryId(16137);
// const storyHistory = await getHistoryByStoryID(16137);

// const miniStats = {
//     story_id: story.id,
//     pulls: story.pull_requests.map( e => {
//         return {
//             pull_id: e.number,
//             url: e.url,
//             repository: e.url
//                 .replace(`https://github.com/Good-Proton/`,'')
//                 .replace(`/pull/${e.number}`,'')
//         }
//     })
// }

// console.log(story);
// console.log(miniStats);
// console.log(getStoryStats(story, storyHistory));


// сохранить в буфере обмена
function saveInExchangeBuffer( string ) {
    const textarea = document.createElement("textarea");
    document.body.appendChild(textarea);
    textarea.value = string;
    textarea.select()
    document.execCommand('copy');
    document.body.removeChild(textarea);
}

// просим данные из буфера обмена
async function askExchangeBuffer() {
    return new Promise( ( res ) => {
        const clickHandler = async () => {
            const text = await getFromExChangeBuffer();
            document.removeEventListener('click', clickHandler);
            res(text);
        }
        
        document.addEventListener('click', clickHandler);
        console.log('Пожалуйста кликните по странице');
        console.log('Скрипт возьмет данные из буфера обмена');
    });
    
    function getFromExChangeBuffer() {
        return new Promise( (res,rej) => {
            const contentEditable = document.createElement("div");
            contentEditable.setAttribute("contenteditable", true);
            
            navigator.clipboard.readText().then( (pastedText) => {
                res(pastedText);
                document.body.removeChild(contentEditable);
            })
            
            document.body.appendChild(contentEditable);
            contentEditable.focus();
            document.execCommand('paste');
        });
}
}


saveInExchangeBuffer('12345')
const res = await askExchangeBuffer();

console.log(res);
```
