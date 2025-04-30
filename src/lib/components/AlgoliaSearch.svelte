<script>
    import { liteClient as algoliasearch } from 'algoliasearch/lite';
    import instantsearch from 'instantsearch.js';
    import { searchBox, hits, configure} from 'instantsearch.js/es/widgets';
    import { connectHits } from 'instantsearch.js/es/connectors';
    import { connectSearchBox, connectStats } from 'instantsearch.js/es/connectors'
    import { pagination } from 'instantsearch.js/es/widgets';
    import {PUBLIC_APP_ID, PUBLIC_ALGOLIA_API_KEY} from '$env/static/public';
    import {onMount} from 'svelte';
    let search;
    onMount(() => {
		//console.log('the component has mounted');

        const searchClient = algoliasearch(PUBLIC_APP_ID, PUBLIC_ALGOLIA_API_KEY);
        let totalRecords;
        let filterQuery;
        // Render the InstantSearch.js wrapper
        // Replace INDEX_NAME with the name of your index.

        search = instantsearch({
            indexName: 'wills',
            searchClient,
        });
        const customHits = connectHits((renderOptions, isFirstRender) => {
            const { hits, widgetParams } = renderOptions;
            const container = widgetParams.container;
            //console.log(hits);
            container.innerHTML = hits.map((hit, index) =>{
                let class_name = index % 2 ? "odd" : "even";
                return `
                    <tr class="${class_name}">
                    <td>${hit.type}</td>
                    <td>${hit._highlightResult.surnames?.value || hit.surnames}</td>
                    <td>${hit._highlightResult.forenames?.value || hit.forenames}</td>
                    <td>${hit._highlightResult.places?.value ||hit.places}</td>
                    <td>${hit._highlightResult.made?.value ||hit.made}</td>
                    <td>${hit._highlightResult.probate?.value || hit.probate}</td>
                    <td>${hit._highlightResult.regref?.value ||hit.regref}</td>
                    <td>${hit._highlightResult.origref?.value || hit.origref}</td>
                    <td>${hit._highlightResult.comments?.value || hit.comments}</td>
                    <td>${hit._highlightResult.year?.value ||hit.year}</td>
                    </tr>`;
            }).join('');
        });

        const customStats = connectStats((renderOptions, isFirstRender) => {
            const {
                nbHits,
                processingTimeMS,
                page,
                nbPages,
                widgetParams,
                query
            } = renderOptions;
            //const query = searchParameters?.query || '';
            if(isFirstRender){
                totalRecords = nbHits;
            }
            let resultsTop = page * 25 + 25 < nbHits ? page * 25 + 25 : nbHits;
            if(query){
                widgetParams.container.innerHTML = `
                    <p>
                    showing  ${page * 25 +1} to ${resultsTop} of ${nbHits} entries (filtered from ${totalRecords} total entries)
                    </p>
                `;
            }else{
                totalRecords = nbHits;
                widgetParams.container.innerHTML = `
                    <p>
                    showing  ${page * 25 +1} to ${resultsTop} of ${nbHits} entries
                    </p>
                `;
            }
            
        });


        search.addWidgets([
            searchBox({
                container: "#searchbox",
            }),

            customHits({
                container: document.querySelector('#hits'),
            }),

            configure({
                hitsPerPage: 25,
            }),
            pagination({
                container: '#pagination',
                showFirst: true,
                showPrevious: true,
                showLast: true,
                showNext: true,
                templates: {
                    first: 'First',
                    previous: 'Previous',
                    next: 'Next',
                    last:'Last'
                }
            }),
            customStats({
                container:  document.querySelector('#text-pagination')
            })
            /*instantsearch.widgets.hits({
                container: "#hits",
                templates: {
                    item: (hit, { html, components }) => html`
                        <tr>
                            <td class="hit-name">
                                ${components.Highlight({ hit, attribute: 'forenames' })}
                            </td>
                            <td class="hit-description">
                                ${components.Highlight({ hit, attribute: 'surnames' })}
                            </td>
                            <td class="hit-price">${hit.year}</td>
                        </tr>
                    `
                },
                cssClasses: {
                    list: '', // removes <ul>
                    item: ''  // removes <li>
                },
                escapeHTML: false 
            })*/
        ]);

        search.start();
        return search;
	});
    function build_query(e){
            e.preventDefault();
            //console.log(e);
            let querys = new Array();
            //let  searchBoxQuery = search.helper.state.query;
            let dateFrom = document.querySelector("#willsTable_range_from_9").value;
            let dateTo = document.querySelector("#willsTable_range_to_9").value;
            let fuzzy = document.querySelector("#willsTable_fuzzy_1").checked;
            let filterParams = {
                type : document.querySelector("select.search_init").value,
                surnames : document.querySelector("#willsTable_text_1").value,
                forenames : document.querySelector("#willsTable_text_2").value,
                places : document.querySelector("#willsTable_text_3").value,
                regref : document.querySelector("#willsTable_text_6").value,
                origref : document.querySelector("#willsTable_text_7").value,
                comments : document.querySelector("#willsTable_text_8").value,
            }
            let queryParameter = {}
            queryParameter.query = search.helper.state.query;
            if(fuzzy){
                queryParameter.restrictSearchableAttributes = ['surnames'];
                queryParameter.query = filterParams.surnames;
                filterParams.surnames = null;
            }
            querys.push(formatDateString(dateFrom, dateTo));
            for(let key in filterParams){
                if(filterParams[key]){
                    let query = `${key}:"${filterParams[key]}"`;
                    querys.push(query);
                }
            }
            querys = querys.filter(item => item);
            //console.log(querys);
            let filterQuery = querys.join(" AND ");
            queryParameter.filters = filterQuery;
            //console.log(filterQuery, queryParameter);
            search.helper
                .setQuery(queryParameter.query)
                .setQueryParameter("filters", queryParameter.filters)
                .setQueryParameter("restrictSearchableAttributes", queryParameter.restrictSearchableAttributes)
                .setQueryParameter("attributesToHighlight", queryParameter.restrictSearchableAttributes)
                .search();
    }
    function formatDateString(dateFrom, dateTo){
        if(dateFrom && dateTo){
            if(!validate_year(dateFrom) && !validate_year(dateTo)){
                return "";
            }
            if(!validate_year(dateTo)){
                return `year <=  ${dateFrom}`;
            }
            if(!validate_year(dateFrom)){
                return `year >=  ${dateTo}`;
            }
            return  `year <=  ${dateFrom} AND year >=  ${dateTo}`;
        }
        if(validate_year(dateFrom)){
            return `year <=  ${dateFrom}`;
        }
        if(validate_year(dateTo)){
            return `year >=  ${dateTo}`;
        }
        return "";
    }
    function validate_year(year_string){
        const regex = new RegExp("^\\d{4}$");
        return regex.test(year_string);
    }
</script>
	

<div class="ais-InstantSearch">
    <div class="right-panel">
        <div class="wrap">
            <div class="table-head"><label class="searchbox-label" for="searchbox">Broad Search:<div id="searchbox"></div></label></div>
            <table>
                <thead>
                    <tr role="row"><th style="width:60px;" class="ui-state-default" role="columnheader" tabindex="0" aria-controls="willsTable" rowspan="1" colspan="1" aria-label="Type: activate to sort column ascending"><div class="DataTables_sort_wrapper">Type<span class="DataTables_sort_icon css_right ui-icon ui-icon-carat-2-n-s"></span></div></th><th style="width:140px;" class="ui-state-default" role="columnheader" tabindex="0" aria-controls="willsTable" rowspan="1" colspan="1" aria-label="Surnames: activate to sort column ascending"><div class="DataTables_sort_wrapper">Surnames<span class="DataTables_sort_icon css_right ui-icon ui-icon-triangle-1-n"></span></div></th><th style="width:100px;" class="ui-state-default" role="columnheader" tabindex="0" aria-controls="willsTable" rowspan="1" colspan="1" aria-label="Forenames: activate to sort column ascending"><div class="DataTables_sort_wrapper">Forenames<span class="DataTables_sort_icon css_right ui-icon ui-icon-triangle-1-n"></span></div></th><th style="width:100px;" class="ui-state-default" role="columnheader" tabindex="0" aria-controls="willsTable" rowspan="1" colspan="1" aria-label="Places: activate to sort column ascending"><div class="DataTables_sort_wrapper">Places<span class="DataTables_sort_icon css_right ui-icon ui-icon-triangle-1-n"></span></div></th><th style="width:50px;" class="ui-state-default" role="columnheader" tabindex="0" aria-controls="willsTable" rowspan="1" colspan="1" aria-label="Made: activate to sort column ascending"><div class="DataTables_sort_wrapper">Made<span class="DataTables_sort_icon css_right ui-icon ui-icon-carat-2-n-s"></span></div></th><th style="width:60px;" class="ui-state-default" role="columnheader" tabindex="0" aria-controls="willsTable" rowspan="1" colspan="1" aria-label="Probate: activate to sort column ascending"><div class="DataTables_sort_wrapper">Probate<span class="DataTables_sort_icon css_right ui-icon ui-icon-carat-2-n-s"></span></div></th><th style="width: 125px;" class="ui-state-default" role="columnheader" tabindex="0" aria-controls="willsTable" rowspan="1" colspan="1" aria-label="Register ref.: activate to sort column ascending"><div class="DataTables_sort_wrapper">Register ref.<span class="DataTables_sort_icon css_right ui-icon ui-icon-carat-2-n-s"></span></div></th><th style="width: 120px;" class="ui-state-default" role="columnheader" tabindex="0" aria-controls="willsTable" rowspan="1" colspan="1" aria-label="Original ref.: activate to sort column ascending"><div class="DataTables_sort_wrapper">Original ref.<span class="DataTables_sort_icon css_right ui-icon ui-icon-carat-2-n-s"></span></div></th><th style="width:130px;" class="ui-state-default" role="columnheader" tabindex="0" aria-controls="willsTable" rowspan="1" colspan="1" aria-label="Comments: activate to sort column ascending"><div class="DataTables_sort_wrapper">Comments<span class="DataTables_sort_icon css_right ui-icon ui-icon-carat-2-n-s"></span></div></th><th style="width:50px;" class="ui-state-default" role="columnheader" tabindex="0" aria-controls="willsTable" rowspan="1" colspan="1" aria-label="Year: activate to sort column ascending"><div class="DataTables_sort_wrapper">Year<span class="DataTables_sort_icon css_right ui-icon ui-icon-carat-2-n-s"></span></div></th></tr>
                    <tr role="row">
                        <th style="width:60px;" class="ui-state-default" rowspan="1" colspan="1"><span class="filter_column filter_select"><select oninput="{build_query}" class="search_init select_filter"><option value="" class="search_init">All</option><option value="Inv">Inv</option><option value="Will">Will</option><option value="Act">Act</option></select></span></th>
                        <th style="width:140px;" class="ui-state-default" rowspan="1" colspan="1"><span class="filter_column filter_text"><input oninput="{build_query}" id="willsTable_text_1" type="text" placeholder="Surnames" class="search_init text_filter fuzzy_search" value=""><span>Fuzzy:</span><input class="fuzzybox" id="willsTable_fuzzy_1" type="checkbox" value="1"></span></th>
                        <th style="width:100px;" class="ui-state-default" rowspan="1" colspan="1"><span class="filter_column filter_text"><input oninput="{build_query}" id="willsTable_text_2" type="text" placeholder="Forenames" class="search_init text_filter" value=""></span></th>
                        <th style="width:100px;" class="ui-state-default" rowspan="1" colspan="1"><span class="filter_column filter_text"><input oninput="{build_query}" id="willsTable_text_3" type="text" placeholder="Places" class="search_init text_filter" value=""></span></th>
                        <th style="width:50px;" class="ui-state-default" rowspan="1" colspan="1">Made</th><th style="width:60px;" class="ui-state-default" rowspan="1" colspan="1">Probate</th>
                        <th style="width:110px;" class="ui-state-default" rowspan="1" colspan="1"><span class="filter_column filter_text"><input oninput="{build_query}" id="willsTable_text_6" type="text" placeholder="Register ref." class="search_init text_filter" value=""></span></th>
                        <th style="width:110px;" class="ui-state-default" rowspan="1" colspan="1"><span class="filter_column filter_text"><input oninput="{build_query}" id="willsTable_text_7" type="text" placeholder="Original ref." class="search_init text_filter" value=""></span></th>
                        <th style="width:130px;" class="ui-state-default" rowspan="1" colspan="1"><span class="filter_column filter_text"><input oninput="{build_query}" id="willsTable_text_8" type="text" placeholder="Comments" class="search_init text_filter" value=""></span></th>
                        <th style="width:50px;" class="ui-state-default" rowspan="1" colspan="1"><span class="filter_column filter_number_range">From <input oninput="{build_query}" type="text" placeholder="Year (e.g. 1621)" inputmode="numeric" pattern="\d{0,4}" maxlength="4" id="willsTable_range_from_9" rel="9"> To <input type="text" inputmode="numeric" pattern="\d{0,4}" placeholder="Year (e.g. 1629)" maxlength="4" class="number_range_filter" id="willsTable_range_to_9" rel="9"></span></th></tr>
                </thead>
                <tbody id="hits">
                </tbody>
            </table>
            <div class="table-base">
                <div id="text-pagination"></div>
                <div id="pagination"></div>
            </div>
        </div>
    </div>
</div>
<style>
    :global .even{
        background-color: white;
    }
    :global .odd{
        background-color: #EEEEEE;
    }
    :global .table-base{
        border: 1px solid #99CCCC;
        background:#d9ecec url("/ui-bf_bright.png") 50% 50% repeat-x;
        color: #333333;
        padding: 5px;
        border-bottom-right-radius: 6px;
        border-bottom-left-radius: 6px;
        display: flex;
        justify-content: space-between;
        padding:5px 4%;
    }
    :global .searchbox-label{
        display: inline-flex;
        color: #009999;
        font-weight: bold;
    }
    :global .searchbox-label div{
        padding-left: 0.5rem;
    }
    :global .table-base #text-pagination p{
        margin: 0;
        color: #009999;
        font-weight: bold;
    }
    :global .table-head{
        border: 1px solid #99CCCC;
        background:#d9ecec url("/ui-bf_bright.png") 50% 50% repeat-x;
        color: #333333;
        padding: 5px;
        border-top-right-radius: 6px;
        border-top-left-radius: 6px;
        padding: 6px;
        text-align: center;
    }
    :global .table-base #text-pagination{
        display: flex;
        align-items: center;
    }
    :global #pagination .ais-Pagination-list{
        display: flex;
        list-style: none;
        margin: 0;
    }
    :global #pagination .ais-Pagination-list .ais-Pagination-item{
        background: #009999 url('/ui-bf_highlight-soft.png') 50% 50% repeat-x;
        margin: 0;
        cursor: pointer;
        border: 1px solid #009999;
        font-weight: bold;
        color: #ffffff;
    }
    :global table tr th{
        background: #009999 url('/ui-bf_highlight-soft.png') 50% 50% repeat-x;
        padding: 5px 10px 6px;
        margin: 0;
        cursor: pointer;
        border: 1px solid #009999;
        font-weight: bold;
        color: #ffffff;
    }
    :global #pagination .ais-Pagination-list .ais-Pagination-link{
        color: #ffffff;
        text-decoration: none;
        width: 100%;
        padding: 5px 10px 6px;
        display: block;
    }
    :global #pagination .ais-Pagination-item.ais-Pagination-item--firstPage
    {
        border-top-left-radius: 6px;
        border-bottom-left-radius: 6px;
    }
    :global #pagination .ais-Pagination-item.ais-Pagination-item--lastPage
    {
        border-top-right-radius: 6px;
        border-bottom-right-radius: 6px;
    }
    :global .wrap{
        max-width: 100%;
        min-width: 768px;
        margin: 0 auto;
        width: 100%;
        overflow-x: auto;
        -webkit-overflow-scrolling: touch; /* Smooth scrolling on iOS */
    }
    :global .wrap table{
       width:100%;
       border: 1px solid #9CC;
    }
    :global th span{
        display: flex;
        overflow: auto;
        flex-flow: column-reverse;
    }
    :global th span{
        display: flex;
        overflow: auto;
        flex-flow: column-reverse;
    }

  :global table {
    width: 100%;
    border-collapse: collapse;
    min-width: 600px;
    table-layout: fixed;
    font-size: 0.75rem;
  }

 th, td {
    padding: 0.5rem;
    text-align: left;
    white-space: nowrap; /* Prevent wrapping */
  }

  :global th {
    background-color: #f9f9f9;
    position: sticky;
    top: 0;
    width: 10% !important;
  }

  tr.even {
    background-color: #fff;
  }

  tr.odd {
    background-color: #f2f2f2;
  }

  @media (max-width: 600px) {
    th, td {
      font-size: 0.875rem;
      padding: 0.5rem;
    }
  }
  @media (min-width: 1920px) {
    :global .wrap{
        width: 66%;
      }
      :global th{
        width: auto !important;
      }
  }
  @media (min-width: 1300px) {
    :global table{
        font-size: 1rem;
      }
  }
</style>