```js
const FETCH_CONFIG = {
    headers:{
        "Tenant-Organization2": "5c916c11-d2f0-4c6b-aa34-be3b41942057",
        "Tenant-Workspace2": "5c9176af-ad29-45f1-94f0-7e5e1d9491c0",
        "Content-Type": "application/json"
    }
};
const LOCAL_STORAGE_KEY = 'collecting_statistics';

const excludeEditorsList = [
    'Alla Sitnikova',
];

if ( new URL(location.href).origin === 'https://app.shortcut.com' ) {
    window.customFieldsJson = await (await fetch('https://app.shortcut.com/backend/api/private/custom-fields',FETCH_CONFIG)).json();
    window.membersListJson = await (await fetch('https://app.shortcut.com/backend/api/private/members',FETCH_CONFIG)).json();
    window.epics = await (await fetch('https://app.shortcut.com/backend/api/private/epics', FETCH_CONFIG)).json();
}

const sum = (...numbers) => {
    return numbers.reduce((total, number) => total + number, 0);
};
const isObject = ( mayBeObject ) => {
    return typeof mayBeObject === 'object' && mayBeObject !== null;
};

async function getHistoryByStoryID(id){
    let res = await fetch(`https://app.shortcut.com/backend/api/private/stories/${id}/history`, FETCH_CONFIG);
    return await res.json();
}
async function getStoryObjectByStoryId(id) {
    const res = await fetch(`https://app.shortcut.com/backend/api/private/stories/${id}`, FETCH_CONFIG );
    return await res.json();
}

// функция для получения массива людей списывающих
// определенные кастомные поля из истории (shortcut)
function getSpandings(storyHistory, fieldNames) {
    let actualValue = 0;
    
    const changes = storyHistory
        .filter( o => o.references)
        .filter( obj => obj.references
            .find( elem => fieldNames.includes(elem.field_name)))
        .map ( obj => {
            const member = getMemberNameById(obj.member_id);
            const references = obj.references
                .filter( o => fieldNames.includes(o.field_name))
            
            let countChanges;
                
            if (references.length === 1) {
                countChanges = parseInt(references[0].string_value) - actualValue;
            } else {
                countChanges = parseInt(references[0].string_value) - parseInt(references[1].string_value);
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

// будем искать название эпика
function getEpicNameById(id) {
    const neededEpic = window.epics.find( obj => obj.id === id);
    return neededEpic ? neededEpic.name : null;
}

// преобразования имен
function convertShortcutMemberName( str ){
    const dictionary = {
        [`Timur Dondokov`]: '', 
        [`Erdem Nikolaev`]: '',
        [`Elvira Baldanova`]: '',
        [`Iaroslav Dzhurov`]: '',
        [`Dolgora Budaeva`]: '', //5

        [`Anne Zolotova`]: 'Zolotova Anne',
        [`Vladimir Mikhno`]: '',
        [`Nikolay Derkachev`]: '',
        [`Anton Leontyuk`]: '',
        [`Grigory Mizhidon`]: '', //10

        [`Sergey Kulabukhov`]: 'Sergey Kulakbuhov',
        [`Alexander Erin`]: '',
        [`Vyacheslav Dorzhiev`]: '',
        [`Daria Lipinskaia`]: '',
        [`Konstantin Solovev`]: '', //15

        [`Vasily Sentyay`]: '',
        [`Vladimir Fyodorov`]: '',
        [`Georgy Nemtsov`]: '',
        [`Alexey Dudyakov`]: '',
        [`Dmitry Nevzorov"`]: '', //20

        [`Ilya Shlyakhovoy`]: '',
        [`Николай Максимов`]: '',
        [`Aarif Khamdi`]: '',
        [`Nikolay Papakha`]: '',
        [`Dmitry Potapenko`]: '', //25

        [`Никита Былёв`]: '',
        [`Alexey Krotov`]: '',
        [`Daria`]: 'Daria Lipinskaia',
        [`Daniil Kolesnikov`]: '',
        [`andy`]: '', //30

        [`Mikhail Zhdanov`]: '',
        [`Сергей Жуткевич`]: '',
        [`Alexandra Pereverzina`]: '',
        [`Igor Karpov`]: '',
        [`Tatyana Zemtsova`]: '', //35

        [`Sergey Slipchenko`]: '',
        [`Victor Konyukhov`]: '',
        [`Zolotova Anne`]: '',
        [`Margarita Gerasimenko`]: '',
        [`vasuta.eugene`]: '', //40

        [`Erdem`]: 'Erdem Nikolaev',
        [`Olga Nikonchuk`]: 'Ольга Никончук',
        [`Denis Romanov`]: '',
        [`Alla Sitnikova`]: '',
        [`Pavel`]: '', //45

        [`Victor Gordeev`]: '',
        [`Tanat`]: '',
        [`Evgenii`]: '',
        [`Alexander Volkov`]: '',
        [`Ivan Sokolov`]: '', //50

        [`Alexander Cherdak`]: '',
        [`Vladimir Rubanyuk`]: '',
        [`Vadim Karpov`]: '',
        [`Dora Budaeva`]: 'Dolgora Budaeva',
        [`Artur Isaev`]: '', //55

        [`Matvey Alexeev`]: '',
        [`Kuzma Shevelev`]: 'Kuzma Shevelev',
        [`Lavr Prokopiev`]: '',
        [`Mikhail Razumovskiy`]: '',
        [`Evgeny Obraztsov`]: '', //60

    };
    
    return dictionary[String(str)] ? dictionary[String(str)] : str;
}

// функция для сбора статистики с стори (shortcut)
function createStoryStats(story, storyHistory) {
    const stats = {
        story_id: story.id,
        story_name: story.name,
        
        type: [story.story_type[0].toUpperCase(), 
               ...story.story_type.slice(1)
        ].join(''),

        owner: convertShortcutMemberName(getMemberNameById(story.owner_ids.find(() => true))),

        pulls: story.pull_requests.map( e => {
            return {
                pull_id: e.number,
                url: e.url,
                repository: e.url
                    .replace(`https://github.com/Good-Proton/`,'')
                    .replace(`/pull/${e.number}`,'')
            }
        }),

        epic_name: getEpicNameById(story.epic_id)
    };

    stats.actual_dev_spendings = getSpandings(storyHistory, ['Actual', 'Actual dev'])
                .map( ({member, total_hours}) => ({ member: convertShortcutMemberName(member), total_hours}) );
    stats.actual_review_spendings = getSpandings(storyHistory, ['Actual review'])
                .map( ({member, total_hours}) => ({ member: convertShortcutMemberName(member), total_hours}) );
    stats.actual_qa_spendings = getSpandings(storyHistory, ['Actual QA'])
                .map( ({member, total_hours}) => ({ member: convertShortcutMemberName(member), total_hours}) );
    
    stats.owners_from_history = stats.actual_dev_spendings.map( ({member}) => convertShortcutMemberName(member) );
    stats.reviewers_from_history = stats.actual_review_spendings.map( ({member}) => convertShortcutMemberName(member) );

    stats.reviewer = convertShortcutMemberName(getCustomFieldValue( story, 'Reviewer' )) || null;
    stats.qa = convertShortcutMemberName(getCustomFieldValue( story, 'QA' )) || null;

    stats.actual_qa = parseInt(getCustomFieldValue( story, 'Actual QA' )) || null;
    stats.actual_dev = parseInt(getCustomFieldValue( story, 'Actual dev' )) || null;
    stats.actual_review = parseInt(getCustomFieldValue( story, 'Actual review' )) || null;

    stats.pulls_count = story.pull_requests.length;
    stats.pull_links = story.pull_requests.map( e => e.url );

    stats.estimate_last_value = story.estimate || null;
    stats.estimate_qa = parseInt(getCustomFieldValue( story, 'Estimate QA' )) || null;

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
async function findReviewers( inputhtml, pullUrl ) {
    const subst = '<a class="assignee Link--primary css-truncate-target width-fit" data-octo-click="hovercard-link-click" data-octo-dimensions="link_type:self"';
    
    let html = inputhtml;
    let positions = findStringPositions( html, subst);

    if (!positions.length) {
        const html2 = await fetchHtml(`${pullUrl}/suggested-reviewers`);
        if (html2 !== 'Not Found') {
            html = html2;
            positions = findStringPositions( html, subst);
        }
    }

    let result = []
    
    if (positions.length) {
        result = positions.map( index => {
            const textPart = html.substring(index, index + 400);
            
            const pattern = /<span class="css-truncate-target width-fit v-align-middle">(.*?)<\/span>/;
            return textPart.match(pattern)[1];
        });
    } 

    if (result.length) {
        result = Array.from(new Set(result));
    }
    
    return result;
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
                    .match(/>(.+)<\/a><\/strong>commented<ahref=/);
            })[0][1];
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

// функция  получает полное имя по нику пользователя гитхаб (github)
async function fetchNameByNickName(nickname) {
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

// конвертация ников в имена
const convertNicknameToFullName = (() => {
    const dictionary1 = {
        ['erinsasha']: 'Alexander Erin',
        ['sanperrier']: 'Vyacheslav Dorzhiev',
        ['uioqouqwe']: 'Grigory Mizhidon',
        ['qlba']: 'Sergey Kulakbuhov',
        ['dariashade']: 'Daria Lipinskaia ',
        ['antondl2']: 'Anton Leontyuk',
        ['antondl3']: 'Anton Leontyuk',
        ['z1ne2wo']: 'Nikolay Derkachev',
        ['kallenju']: 'Konstantin Solovev',
        ['AtlantaTesla']: 'Zolotova Anne',
        ['vladimir-mikhno']: 'Vladimir Mikhno',
        ['VladimirFyodorov']: 'Vladimir Fyodorov',
        ['unM3RCURRRY']: 'Alexander Cherdak',
        ['espeluznante']: 'Pavel',
        ['prestonviewer']: 'Vladimir Rubanyuk',
        ['lindon0390']: 'Timur Dondokov',
        ['iaroslav-dz']: 'Iaroslav Dzhurov',
        ['Frcjocker']: 'Erdem Nikolaev',
        ['olganikvik']: 'Ольга Никончук',
        ['niki4xxxxx']: 'Никита Былёв',
        ['EllaBaldanova']: 'Elvira Baldanova',
        ['lhadminonduty ']: 'Admin',
        ['AlmadelX']: 'Matvey Alexeev',
        ['kritma']: 'Vasily Sentyay',
        ['DoraBudaeva']: 'Dolgora Budaeva',
        ['Senails']: 'Kuzma Shevelev',
    };
    
    const dictionary2 = {};

    return async( nickname ) => {
        if (dictionary1[nickname]) {
            return dictionary1[nickname];
        }
        if (!dictionary2[nickname]) {
            dictionary2[nickname] = await fetchNameByNickName(nickname);
        }
        return dictionary2[nickname];
    }
})();

// Функция должна брать адрес пула и возвращать данные по нему
async function callectPullData( url ) {
    const pullPage = await fetchHtml(url)

    const ownerPromise = convertNicknameToFullName(findOwner(pullPage));
    const reviewersPromise = Promise
        .all( await findReviewers(pullPage, url).then( reviewers => reviewers
            .map( nick => convertNicknameToFullName(nick))));

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

// ищем стори созданые после какой то даты (shortcut)
async function searchStoryInRange(startDate, endDate) {
    const res = await fetch('https://app.shortcut.com/backend/api/private/stories/search',{
        ...FETCH_CONFIG,
        method: 'POST',
        body: JSON.stringify({
            created_at_start: startDate.toISOString(),
            created_at_end: endDate.toISOString(),
        }),
    })
    
    const array = await res.json();
    return array.map(s => s.id).sort((num1, num2) => num1 - num2 );
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
async function saveAsFile( data, fileName, fileFormat = 'csv') {
    const obj = { data };

    const text = fileFormat === 'csv' 
        ? createCsvText(splitObjects(data))
        : JSON.stringify(obj, null, 2);

    const type = fileFormat === 'csv' 
        ? 'text/csv' 
        : 'application/json';

    const blob = new Blob([text], { type });
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');

    a.href = url;
    a.download = fileName;
    
    document.body.appendChild(a);
    a.click();
    
    document.body.removeChild(a);
    URL.revokeObjectURL(url);
}

// раскладываем вложеный обьект на много плоских
function splitObjects( objects ) {
    return objects
        .map( joinPrimitiveValues )
        .map( convert )
        .flat();
    
    function joinPrimitiveValues( obj ) {
        Object.keys(obj).forEach( key => {
            if ( Array.isArray(obj[key]) && obj[key].length) {
                const array = obj[key];
                if (isObject(array[0])) {
                    obj[key] = array.map(joinPrimitiveValues);
                } else {
                    obj[key] = array.map(v => String(v)).join(', ');
                }
                return;
            }
            
            if ( !Array.isArray(obj[key]) && isObject(obj[key])) {
                obj[key] = joinPrimitiveValues(obj[key]);
            }
        })

        return obj;
    }
    function convert( obj ) {
        Object.keys(obj).forEach( key => {
            if ( Array.isArray(obj[key]) ) {
                return obj[key] = obj[key].map(convert).flat();
            }
            if ( isObject(obj[key]) ) {
                return obj[key] = convert(obj[key]);
            }
        })
      
        let objects = [obj];
        
        Object.keys(obj)
            .filter( k => Array.isArray(obj[k]) && obj[k].length )
            .forEach( key => {
                objects = objects.map( o => o[key]
                    .map( arrElem => ({...o, [key]: arrElem}) ) 
                ).flat();
            });
        
        return objects.map( obj => {
            const newObject = {};

            Object.keys(obj).forEach( key => {
                if (isObject(obj[key])) {
                    const subObj = obj[key];
                    return Object.keys(subObj).forEach( k => {
                        newObject[`${key}_${k}`] = subObj[k];
                    })
                }
                newObject[key] = obj[key];
            })

            return newObject;
        })
    }
}

// формируем csv файл
function createCsvText( data, isNeedHead = true ) {
    const objectStruct = getObjectStruct(data);
    
    const head = createHead({head: objectStruct}, 'head')
        .filter((_, i) => i)
        .map( a => a.join(','))
        .join('\n');
    
    const value = data.map( obj => Object.keys(objectStruct)
        .map( key => createValue(obj, objectStruct, key) ))
        .map( a => a.flat())
        .map( a => a.join(','))
        .join('\n');
    
    return `${ isNeedHead ? `${head}\n`: '' }${value}`;

    function format( inputValue ) {
        const value = inputValue === null ? ''
            : typeof inputValue === 'number' ? String(inputValue).replace('.',',')
            : String(inputValue);
        
        return `"${value.replaceAll('"','""')}"`;
    }
    function getObjectStruct( data ) {
        const struct = {};
        
        data.forEach( elem => Object.keys(elem)
            .forEach( key => {
                if (!struct[key]) {
                    if ( !isObject(elem[key]) ) {
                        return struct[key] = true;
                    }
                    
                    const subStruct = getObjectStruct(data.map( elem => elem[key]).filter( e => e ));
                    if (Object.keys(subStruct).length) {
                        struct[key] = subStruct;
                    }
                }
            }) 
        )
        
        return struct;
    }
    function findDeepChildCount( obj, key) {
        return isObject(obj[key]) 
            ? sum( ...Object.keys(obj[key]).map( k => findDeepChildCount(obj[key], k)) )
            : 1 ;
    }
    function createHead( obj, key, parentName = '') {
        const formatKey = obj[0] ? format(`${parentName}[${key}]`) : format(key);
        
        if (!isObject(obj[key])) {
            return [[formatKey]]
        }

        const childStructs = Object.keys(obj[key])
            .map( k => createHead(obj[key], k, key))
            .map( (arr, _, all) => {
                const maxLenth = Math.max( ...all.map( e => e.length ) );
                if (arr.length < maxLenth) {
                    const lengthArrays = arr[0].length;
                    arr.push( ...Array(maxLenth - arr.length).fill(arr[arr.length - 1]));
                }
                return arr;
            });
        
        const mergedChildrens = childStructs[0].map((_, j) => [ ...childStructs.map((_,i) => childStructs[i][j])].flat() )
        
        return [[formatKey, ...Array(findDeepChildCount(obj, key) - 1).fill(formatKey)], ...mergedChildrens ];
    }
    function createValue( obj, struct, key) {
        if (isObject(struct[key])) {
            const subObject = obj ? obj[key] : null;
            const substruct = struct[key];

            return Object.keys(substruct)
                .map( key => createValue(subObject, substruct, key))
                .flat();
        }
        
        return (obj && obj[key] !== undefined) ? [format(obj[key])] : [format(``)] ;
    }
}

// Шаг1 сбор статистики с шотката (shortcut)
async function step1(storyIds, localStorageData) {
    const allStats = [];

    try {
        for(const id of storyIds) {
            const savedData = localStorageData && localStorageData.find( s => s.story_id === id);
            const storyStats = savedData || await getStoryStatsById(id);
            
            allStats.push(storyStats);
            console.log(`Сибираем статистику по сторям ${allStats.length} / ${storyIds.length} story_id = ${id}`);
        }
        return {
            isSucces: true,
            data: allStats,
        }
    } catch(e) {
        console.error(e);
        return {
            isSucces: false,
            data: allStats,
        }
    }
}

// Шаг2 сбор статистики с гитхаба (github)
async function step2(inputData, localStorageData) {
    const outputData = [];
    let i = 1;

    try {
        for( const statsStory of inputData) {
            const savedData = localStorageData && localStorageData
                .find( s => s.story_id === statsStory.story_id);
            const storyStats = savedData || {
                ...statsStory,
                pulls: await Promise.all(statsStory.pulls
                    .map( pull => callectPullData(pull.url)
                        .then( data => ({ ...pull, ...data }))))
            };
            
            outputData.push(storyStats);
            console.log(`Собираем данные по пулам для каждой стори ${i++} / ${inputData.length} story_id = ${statsStory.story_id}`);
        }

        return {
            isSucces: true,
            data: outputData,
        }
        
    } catch(e) {
        console.error(e);
        return {
            isSucces: false,
            data: outputData,
        }
    }
}

// Шаг3 обьеденение статистик (shortcut)
function step3(data1, data2) {
    if (data1.length !== data2.length) {
        throw new Error();
    }
    
    return data1.map( (strory, i) => ({...strory, ...data2[i] }));
}

async function collectStats( createDateStr, inDevDateStr, endRangeDate, fileFormat = 'csv') {
    // получаем данные из хранилища и буфера обмена
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
        const inDevDate = new Date(inDevDateStr);
        const createDateStart = new Date(createDateStr);
        const createDateEnd = new Date(endRangeDate);
        
        const arrayStoryIds = await searchStoryInRange(createDateStart, createDateEnd);
        const unicString = createDateStr + inDevDateStr + endRangeDate;
        
        if (
            localStorageData
            && localStorageData.step === 3
            && localStorageData.isSucces
            && localStorageData.unicString === unicString
        ) {
            const finalData = localStorageData.data
                // .filter((_, i) => i< 6);
            
            console.log('Данные сохранены в локальном хранилище');
            console.log(finalData);

            await askExchangeBuffer('Кликниете на странице, скрипт сохранит статистику как файл');
            return await saveAsFile(finalData, 'stats', fileFormat);
        }

        if (
            exchangeBufferData
            && localStorageData
            && exchangeBufferData.step === 2
            && localStorageData.step === 1
            && exchangeBufferData.isSucces
            && localStorageData.isSucces
            && exchangeBufferData.unicString === localStorageData.unicString
        ) {
            const localData = localStorageData.data
                .filter( stats => new Date(stats.first_move_to_in_development) - inDevDate > 0);
            const step3Data = step3(localData, exchangeBufferData.data);
            const finalData = step3Data.map( storyStats => ({
                    ...storyStats,
                    max_qa_rejected_count: Math.max( ...storyStats.pulls.map( pull => pull.qa_rejected_count), ...[0]),
                    total_reviewer_rejected_count: sum( ...storyStats.pulls.map( pull => pull.reviewer_rejected_count)),
                }));
           
            localStorage.setItem(LOCAL_STORAGE_KEY, JSON.stringify({
                step: 3,
                isSucces: true,
                data: finalData,
                unicString: localStorageData.unicString,
            }))
            
            console.log('Данные сохранены в локальном хранилище');
            console.log(finalData);

            await askExchangeBuffer('Кликниете на странице, скрипт сохранит статистику как файл');
            return await saveAsFile(finalData, 'stats', fileFormat);
        }
        
        if (
            !exchangeBufferData
            && localStorageData
            && localStorageData.step === 1
            && localStorageData.isSucces
            && localStorageData.unicString === unicString
        ) {
            const dataForBuffer = localStorageData;
            dataForBuffer.data = dataForBuffer.data.map( story => ({
                story_id: story.story_id,
                pulls: story.pulls
            }));
            
            saveInExchangeBuffer(JSON.stringify(dataForBuffer));
            return console.log('Можно перейти на страницу гитхаба и запустить скрипт там');
        } 

        if (
            !localStorageData
            || !localStorageData.isSucces
            || localStorageData.unicString !== unicString
        ) {
            const savedData = 
                localStorageData 
                && localStorageData.unicString === unicString
                && localStorageData.data;
            
            const {isSucces, data} = await step1(arrayStoryIds, savedData);

            const save = {
                    step: 1,
                    isSucces,
                    data,
                    unicString,
                }

            localStorage.setItem(LOCAL_STORAGE_KEY, JSON.stringify(save))

            if (isSucces) {
                save.data = save.data
                    .filter( stats => new Date(stats.first_move_to_in_development) - inDevDate > 0);
                
                save.data = save.data.map( ({story_id, pulls}) => ({story_id, pulls}));

                await askExchangeBuffer('Кликните пожалуйста по странице, это даст скрипту доступ к буферу обмена');
                saveInExchangeBuffer(JSON.stringify(save));

                console.log('Можно перейти на страницу гитхаба и запустить скрипт там, для продолжения сбора статистики');
            } else {
                console.log('Что то пошло не так, если эта ошибка сети то при повторном запуске скрипт попробует продолжить, а не начинать с нуля');
            }
            return;
        }

        if (
            exchangeBufferData
            && exchangeBufferData
            && exchangeBufferData.step === 1
            && exchangeBufferData.isSucces
            && exchangeBufferData.unicString === unicString
        ) {
            return console.log('Данные уже в буфере обмена')
        }
    }

    if ( url === 'https://github.com') {
        if (
            exchangeBufferData 
            && localStorageData
            && exchangeBufferData.step === 1
            && exchangeBufferData.isSucces
            && localStorageData.isSucces
            && exchangeBufferData.unicString === localStorageData.unicString
        ) {
            saveInExchangeBuffer(JSON.stringify(localStorageData));
            
            console.log('Можно вернуться в шоткат и обьеденить данные повторно запустив скрипт');
            return;
        } 

        if (
            exchangeBufferData 
            && exchangeBufferData.step === 1
            && exchangeBufferData.isSucces
        ) {
            const savedData = 
                localStorageData 
                && localStorageData.unicString === exchangeBufferData.unicString
                && localStorageData.data;
            
            console.log('Начинаем сбор данных, это может занять какое то время');
            const { isSucces, data } =  await step2( exchangeBufferData.data, savedData);
            
            const save = {
                step: 2,
                isSucces,
                data,
                unicString: exchangeBufferData.unicString,
            }
            
            const text = JSON.stringify(save);
            localStorage.setItem(LOCAL_STORAGE_KEY, text)
            if (isSucces) {
                await askExchangeBuffer('Кликните пожалуйста по странице, это даст скрипту доступ к буферу обмена');
                saveInExchangeBuffer(text);
                
                console.log('Можно вернуться в шоткат и обьеденить данные повторно запустив скрипт');
            } else {
                console.log('Что то пошло не так, если эта ошибка сети то при повторном запуске скрипт попробует продолжить, а не начинать с нуля');
            }
            return;
        }

        console.log('похоже в буфере обмена пусто, соберите сначало данные с шотката')
    }
}

// стори попадает в статистику если:
// 1. она создана после первой даты
// 2. была перемещена в разработку после второй даты
// 3. была создана до 3 даты

await collectStats('2021.06.02', '2023.01.01', '2024.01.01');
// await collectStats('2023.10.02', '2023.12.01', '2024.01.01');
```
