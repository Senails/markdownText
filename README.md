```js
const FETCH_CONFIG = {
    headers:{
        "Tenant-Organization2": "5c916c11-d2f0-4c6b-aa34-be3b41942057",
        "Tenant-Workspace2": "5c9176af-ad29-45f1-94f0-7e5e1d9491c0",
        "Content-Type": "application/json"
    }
};
const LOCAL_STORAGE_KEY = 'collecting_statistics';

if ( new URL(location.href).origin === 'https://app.shortcut.com' ) {
    window.customFieldsJson = await (await fetch('https://app.shortcut.com/backend/api/private/custom-fields',FETCH_CONFIG)).json();
    window.membersListJson = await (await fetch('https://app.shortcut.com/backend/api/private/members',FETCH_CONFIG)).json();
}

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
// определенные кастомные поля из истории (shortcut)
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

// поиск имени по айди пользователя (shortcut)
function getMemberNameById(memberId) {
    const member = membersListJson
        .find( m => m.id === memberId);

    if (member) {
        return member.profile.name;
    }
    return memberId;
}

// получаем значение поля по его имени (shortcut)
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
// в определенном состоянии (shortcut)
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

// поиск разницы в днях между датами (shortcut)
function getDaysBetweenDates( dateIso1 , dateIso2) {
    if (dateIso1) {
        const date1 = new Date(dateIso1);
        const date2 = new Date(dateIso2 || Date.now());

        return Math.floor((date2 - date1)/(60*60*24))/1000;
    }
    return null;
}

// функция для сбора статистики с стори (shortcut)
function createStoryStats(story, storyHistory) {

    const stats = {
        story_id: story.id,
        story_name: story.name,
        
        type: [story.story_type[0].toUpperCase(), 
               ...story.story_type.slice(1)
        ].join(''),

        owner: getMemberNameById(story.owner_ids.find(() => true)),

        pulls: story.pull_requests.map( e => {
            return {
                pull_id: e.number,
                url: e.url,
                repository: e.url
                    .replace(`https://github.com/Good-Proton/`,'')
                    .replace(`/pull/${e.number}`,'')
            }
        })
    };

    stats.actual_dev_spendings = getSpandings(storyHistory, ['Actual', 'Actual dev']);
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

// функция для сбора статистики с стори по айди (shortcut)
async function getStoryStatsById( storiId ) {
    const story = await getStoryObjectByStoryId(storiId);

    if (!(story.message && story.message === 'Resource not found.')) {
        const storyHistory = await getHistoryByStoryID(storiId);
        return createStoryStats(story, storyHistory);
    }
    
    return;
}

// В гитхабе все данные подгружается в виде html кусков
// поэтому поиск осуществляю так же загружая html, и проводя поиск в коде

// получаем html код по ссылке (github)
async function fetchHtml( url ) {
    return await (await fetch(url)).text();
}

// регуляркой ищет позиции вложений подстроки (github)
function findStringPositions( str, substr ) {
    const regexPattern = new RegExp(substr,'g');
    const found = Array.from(str.matchAll(regexPattern));
    return found.map( elem => elem.index );
}

// ищем количество добавлений тега QA Rejected в html коде (github)
function findQARejectCount( html ) {
    const substring = `data-name="QA Rejected" style="--label-r:217;--label-g:63;--label-b:11;--label-h:15;--label-s:90;--label-l:44;" data-view-component="true" class="IssueLabel hx_IssueLabel d-inline-block v-align-middle">`;
    const positions = findStringPositions( html, substring);

    return positions.map( index => {
        const textPart = html.substring(index - 150, index);
        const cleanedText = textPart.replaceAll(' ','').replaceAll('\n','');
        
        const sbstr1 = `</a>added<aid="label-`;
        const sbstr2 = `</a>addedthe<aid="label-`;
        return cleanedText.includes(sbstr1) || cleanedText.includes(sbstr2);
    }).filter( r => r).length;
}

// ищем количество реджектов от ревьювера в html коде (github)
function findReviewerRejectCount( html ) {
    const substring = `requested changes`;
    const positions = findStringPositions( html, substring);

    if (positions.length) {
        return positions.map( index => {
            const textPart = html.substring(index - 1200, index + 50);
            
            const substr = '<path d="M1 1.75C1 .784 1.784 0 2.75 0h7.586c.464 0 .909.184 1.237.513l2.914 2.914c.329.328.513.773.513 1.237v9.586A1.75 1.75 0 0 1 13.25 16H2.75A1.75 1.75 0 0 1 1 14.25Zm1.75-.25a.25.25 0 0 0-.25.25v12.5c0 .138.112.25.25.25h10.5a.25.25 0 0 0 .25-.25V4.664a.25.25 0 0 0-.073-.177l-2.914-2.914a.25.25 0 0 0-.177-.073ZM8 3.25a.75.75 0 0 1 .75.75v1.5h1.5a.75.75 0 0 1 0 1.5h-1.5v1.5a.75.75 0 0 1-1.5 0V7h-1.5a.75.75 0 0 1 0-1.5h1.5V4A.75.75 0 0 1 8 3.25Zm-3 8a.75.75 0 0 1 .75-.75h4.5a.75.75 0 0 1 0 1.5h-4.5a.75.75 0 0 1-.75-.75Z"></path>';
            const substr2 = '<tool-tip id="tooltip';
            
            return textPart.includes(substr) && !textPart.includes(substr2);
        }).filter( c => c).length;
    }
    return 0;
}

// Поиск ревьюверов (github)
function findReviewers( html ) {
    const subst = '<a class="assignee Link--primary css-truncate-target width-fit" data-octo-click="hovercard-link-click" data-octo-dimensions="link_type:self"';
    const positions = findStringPositions( html, subst);

    if (positions.length) {
        return positions.map( index => {
            const textPart = html.substring(index, index + 400);
            
            const pattern = /<span class="css-truncate-target width-fit v-align-middle">(.*?)<\/span>/;
            return textPart.match(pattern)[1];
        });
    }
    return [];
}

// Поиск создателя пула (github)
function findOwner( html ) {
    const subst = '<a class="author Link--primary text-bold css-overflow-wrap-anywhere " show_full_name="false" data-hovercard-type="user" data-hovercard-url=';
    const positions = findStringPositions( html, subst);

    if (positions.length) {
        return positions
            .map( index => html.substring(index, index + 450))
            .filter( text => text.includes('commented'))
            .map( textPart => {
                return textPart
                    .replaceAll(' ','')
                    .replaceAll('\n','')
                    .match(/>(.+)<\/a><\/strong>commented<ahref=/)[1];
            })[0]
    }
    return;
}

// в гитхабе история пула сворачивается и есть кнопка "Load more"
// в ней есть url по ней можно получить следующий кусок html (github)
function findNextUrls( html ) {
    const substring = '<button type="submit" class="ajax-pagination-btn no-underline pb-1 pt-0 px-4 mt-0 mb-1 color-bg-default border-0" data-disable-with="Loading…">';
    const positions = findStringPositions( html, substring);
    
    if (positions.length) {
        return positions.map( index => {
            const textPart = html
                .substring(index - 750, index - 200);

            const regExp = /action="[^"]+"/;
            return textPart.match(regExp)[0]
                .replace('action="','https://github.com')
                .replace('"','');
        })
    }
    return [];
}

// функция будет получать полное имя по нику пользователя гитхаб (github)
async function convertNicknameToFullName(nickname) {
    const substr = '<span class="p-name vcard-fullname d-block overflow-hidden" itemprop="name">'
    const html = await fetchHtml(`https://github.com/${nickname}`);

    const index = findStringPositions(html, substr)[0];
    const textPart = html.substring(index, index + 250);

    const fullName = textPart
    .replaceAll('\n','')
    .match(/<span class="p-name vcard-fullname d-block overflow-hidden" itemprop="name">([^<]+)<\/span> /)[1]
    .replace(/^ +/,'').replace(/ +$/,'');

    return fullName || nickname;
}

// Функция должна брать адрес пула и возвращать обьект
// В обьекте должны присутствовать поля:
// owner, reviwers, qa_rejected_count, reviewer_rejected_count (github)
async function callectPullData( url ) {
    const pullPage = await fetchHtml(url)

    const ownerPromise = convertNicknameToFullName(findOwner(pullPage));
    const reviewersPromise = Promise.all(findReviewers(pullPage).map( nick => convertNicknameToFullName(nick)));

    let htmls = [pullPage];
    let qa_rejected_count = 0;
    let reviewer_rejected_count = 0;

    while (htmls.length) {
        const nextUrls = [];
        
        htmls.forEach( html => {
            qa_rejected_count += findQARejectCount(html);
            reviewer_rejected_count += findReviewerRejectCount(html);
            nextUrls.push( ...findNextUrls(html) )
        })

        htmls = !nextUrls.length ? [] 
            : await Promise.all(nextUrls.map( url => fetchHtml(url) ));
    }
    
    return {
        owner: await ownerPromise,
        reviewers: await reviewersPromise,
        qa_rejected_count,
        reviewer_rejected_count,
    };
}

// сохранить в буфере обмена
function saveInExchangeBuffer( string ) {
    const textarea = document.createElement("textarea");
    document.body.appendChild(textarea);
    textarea.value = string;
    textarea.select()
    document.execCommand('copy');
    document.body.removeChild(textarea);
}

// получаем из в буфера обмена
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

// просим пользователя кликнуть, что бы получить доступ
async function askExchangeBuffer(message) {
    return new Promise( ( res ) => {
        const clickHandler = async () => {
            const text = await getFromExChangeBuffer();
            document.removeEventListener('click', clickHandler);
            res(text);
        }
        
        document.addEventListener('click', clickHandler);
        console.log(message);
    });
}

// сохраняем статистику в файл
async function saveAsFile( data, fileName) {
    const obj = { data };
    const blob = new Blob([JSON.stringify(obj)], { type: 'application/json' });
    
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    
    a.href = url;
    a.download = fileName;
    
    document.body.appendChild(a);
    a.click();
    document.body.removeChild(a);
    URL.revokeObjectURL(url);
}

// Шаг1 сбор статистики с шотката (shortcut)
async function step1(rangeStart, rangeEnd, saveData) {
    const allStats = [];
    let actualId = rangeStart;

    try {
        while ( actualId <= rangeEnd) {
            const storyStats = await getStoryStatsById(actualId);
            if (storyStats) {
                allStats.push(storyStats);
            }
    
            console.log(`Сибираем статистику по сторям ${actualId - rangeStart +1} / ${rangeEnd - rangeStart +1}`);
            actualId++;
        }

        await saveData(true, allStats);
    } catch {
        await saveData(false, allStats);
    }
}

// Шаг2 сбор статистики с гитхаба (github)
async function step2(inputData, saveData) {
    const outputData = [];
    let i = 1;

    try {
        for( const statsStory of inputData) {
            let obj = {
                pulls: await Promise.all(statsStory.pulls.map( pull => {
                    return new Promise( async (res) => {
                        res({
                            ...pull,
                            ...(await callectPullData(pull.url))
                        });
                    })
                }))
            };
            outputData.push(obj);
            console.log(`Собираем данные по пулам для каждой стори ${i++} / ${inputData.length}`);
        }
        await saveData(true, outputData);
    } catch(e) {
        console.log(e);
        await saveData(false, outputData);
    }
}

// Шаг3 обьеденение статистик (shortcut)
function step3(data1, data2) {
    if (data1.length !== data2.length) {
        throw new Error();
    }
    
    return data1.map( (strory, i) => ({...strory, ...data2[i] }));
}

async function collectStatsInRange(rangeStart, rangeEnd) {
    const localStorageDataJson = localStorage.getItem(LOCAL_STORAGE_KEY);
    await askExchangeBuffer('Кликните пожалуйста по странице, это даст скрипту доступ к буферу обмена');
    const exchangeBufferDataJson = await getFromExChangeBuffer();
    
    let localStorageData = localStorageDataJson && JSON.parse(localStorageDataJson) || null;
    let exchangeBufferData;

    try {
        exchangeBufferData = JSON.parse(exchangeBufferDataJson);
    } catch {
        exchangeBufferData = null
    }

    const url = new URL(location.href).origin;
    
    if ( url === 'https://app.shortcut.com' ) {
        if (
            localStorageData
            && localStorageData.step === 3
            && localStorageData.isSucces
            && localStorageData.rangeStart === rangeStart
            && localStorageData.rangeEnd === rangeEnd
        ) {
            const finalData = localStorageData.data;
            
            console.log('Данные сохранены в локальном хранилище');
            console.log(finalData);

            await askExchangeBuffer('Кликниете на странице, скрипт сохранит статистику как файл');
            await saveAsFile(finalData, 'stats');
        }

        if (
            exchangeBufferData
            && localStorageData
            && exchangeBufferData.step === 2
            && localStorageData.step === 1
            && exchangeBufferData.isSucces
            && localStorageData.isSucces
            && exchangeBufferData.rangeStart === localStorageData.rangeStart
            && exchangeBufferData.rangeEnd === localStorageData.rangeEnd
        ) {
            const finalData = step3(localStorageData.data, exchangeBufferData.data);
            localStorage.setItem(LOCAL_STORAGE_KEY, JSON.stringify({
                step: 3,
                isSucces: true,
                data: finalData,
                rangeStart,
                rangeEnd,
            }))
            
            console.log('Данные сохранены в локальном хранилище');
            console.log(finalData);

            await askExchangeBuffer('Кликниете на странице, скрипт сохранит статистику как файл');
            await saveAsFile(finalData, 'stats');
        }

        if (
            !exchangeBufferData
            && localStorageData
            && localStorageData.step === 1
            && localStorageData.isSucces
            && localStorageData.rangeStart === rangeStart
            && localStorageData.rangeEnd === rangeEnd
        ) {
            const dataForBuffer = localStorageData;
            dataForBuffer.data = dataForBuffer.data.map( story => ({
                story_id: story.story_id,
                pulls: story.pulls
            }));

            
            saveInExchangeBuffer(JSON.stringify(dataForBuffer));
            console.log('все хорошо, данные с шотката собраны, и сохранены в локальном хранилище');
            console.log('можно перейти на страницу гитхаба и запустить скрипт там');
            console.log('https://github.com/Senails/markdownText/blob/full-stats-script/README.md');

            return;
        } 

        if (
            !localStorageData
            || localStorageData.rangeStart !== rangeStart
            || localStorageData.rangeEnd !== rangeEnd
        ) {
            await step1 (rangeStart, rangeEnd, async (isSucces, data) => {
                saveInExchangeBuffer('');
                const save = {
                    step: 1,
                    isSucces,
                    data,
                    rangeStart,
                    rangeEnd,
                }
                
                localStorage.setItem(LOCAL_STORAGE_KEY, JSON.stringify(save))
                if (isSucces) {
                    save.data = save.data.map( story => ({
                        story_id: story.story_id,
                        pulls: story.pulls
                    }))

                    await askExchangeBuffer('Кликните пожалуйста по странице, это даст скрипту доступ к буферу обмена');
                    saveInExchangeBuffer(JSON.stringify(save));
                    
                    console.log('все хорошо, данные с шотката собраны, и сохранены в локальном хранилище, данные для сбора на гитхабе помещены в буфер обмена');
                    console.log('можно перейти на страницу гитхаба и запустить скрипт там, для продолжения сбора статистики');
                    console.log('https://github.com/Senails/markdownText/blob/full-stats-script/README.md');
                } else {
                    console.log('что то пошло не так');
                }
            })
        }
    }

    if ( url === 'https://github.com') {
        if (
            exchangeBufferData 
            && localStorageData
            && exchangeBufferData.step === 1
            && exchangeBufferData.isSucces
            && exchangeBufferData.rangeStart === localStorageData.rangeStart
            && exchangeBufferData.rangeEnd === localStorageData.rangeEnd
        ) {
            saveInExchangeBuffer(JSON.stringify(localStorageData));
            
            console.log('данные с гитхаба сохранены в локальном хранилище и буфере обмена');
            console.log('можно вернуться в шоткат и обьеденить данные повторно запустив скрипт');
            console.log('https://app.shortcut.com/linkedhelper/story/17045/shortcut');
            return;
        } 

        if (
            exchangeBufferData 
            && exchangeBufferData.step === 1
            && exchangeBufferData.isSucces
        ) {
            console.log('начинаем собирать данные');
            return await step2( exchangeBufferData.data, async ( isSucces, data ) => { 
                const save = {
                    step: 2,
                    isSucces,
                    data,
                    rangeStart: exchangeBufferData.rangeStart,
                    rangeEnd: exchangeBufferData.rangeEnd,
                }
                const text = JSON.stringify(save);
                
                localStorage.setItem(LOCAL_STORAGE_KEY, text)
                if (isSucces) {
                    console.log('начинаем сбор данных, это может занять какое то время');
                    await askExchangeBuffer('Кликните пожалуйста по странице, это даст скрипту доступ к буферу обмена');
                    saveInExchangeBuffer(text);
                    
                    console.log('данные с гитхаба сохранены в локальном хранилище и буфере обмена');
                    console.log('можно вернуться в шоткат и обьеденить данные повторно запустив скрипт');
                    console.log('https://app.shortcut.com/linkedhelper/story/17045/shortcut');
                } else {
                    console.log('что то пошло не так');
                }
            })
        }

        console.log('похоже в буфере обмена пусто, соберите сначало данные с шотката')
    }
}

await collectStatsInRange(13306, 13406);

```
