from flask import Flask, jsonify, request, session, render_template, url_for
import json
import urllib3
import pandas as pd
from fuzzywuzzy import fuzz
import time
import asyncio
import aiohttp

application = Flask(__name__)
application.secret_key = ""
a = {}
filtered_morp = []
lock = asyncio.Lock()

@application.route("/question", methods=["POST"])
def get_question():
    loop = asyncio.new_event_loop()
    asyncio.set_event_loop(loop)
    result = loop.run_until_complete(get_question_async())
    return result

async def get_question_async():
    global a
    global generated_text

    if request.method == "POST":
        generated_text = ""

    request_data = json.loads(request.get_data(), encoding='utf-8')
    a[request_data['userRequest']['user']['id']] = '아직 AI가 처리중이에요'
    question = request_data['action']['params']['question']
    question += ' 관련 학과 추천해줘. 답변은 오직 학과명만 나열해. 학과끼리는 쉼표로 구분해줘.'
    chat_info_setting(question)

    try:
        starttime = time.time()

        async with aiohttp.ClientSession() as aiohttp_session:
            async with lock:
                async with aiohttp_session.post(
                    "https://api.openai.com/v1/chat/completions",   # 서버 주소 수정
                    headers=headers,
                    json=data,
                ) as response:
                    response.raise_for_status()
                    response_data = await response.json()
                    generated_text = response_data['choices'][0]['message']['content']
                    session["generated_text"] = generated_text
                    endtime = time.time()
    except aiohttp.ClientError as e:
        print(f"Error occurred: {e}")

    response = {
        "version": "2.0",
        "template": {
            "outputs": [{
                "simpleText": {
                    "text": generated_text + ' 등의 학과를 추천해드리겠습니다.\n\n분야를 세분화하여 전공을 추천해드리려 합니다. 계속하시려면 "예" 를 입력해주세요. '
                }
            }]
        }
    }
    return jsonify(response)

@application.route("/schools", methods=["POST"])
def get_schools():
    global generated_text
    global filtered_morp
    filtered_morp = []

    request_data = json.loads(request.get_data(), encoding='utf-8')
    a[request_data['userRequest']['user']['id']] = '아직 AI가 처리중이에요'

    openApiURL = "http://aiopen.etri.re.kr:8000/WiseNLU"
    accessKey = ""
    analysisCode = "wsd"
    text = generated_text
    requestJson = {
        "argument": {
            "text": text,            "analysis_code": analysisCode
        }
    }
    http = urllib3.PoolManager()
    # starttime = time.time()
    response = http.request(
        "POST",
        openApiURL,
        headers={"Content-Type": "application/json; charset=UTF-8", "Authorization": accessKey},
        body=json.dumps(requestJson, indent=4)
    )

    if response.status != 200:
        print("Error: API request failed. Status code:", response.status)
        return jsonify({"error": "API request failed"})

    try:
        parsed_data = json.loads(response.data)
    except json.decoder.JSONDecodeError as e:
        print("Error: JSON decoding failed.", e)
        return jsonify({"error": "JSON decoding failed"})

    data = response.data

    # JSON 데이터 파싱
    parsed_data = json.loads(data)
    # print(parsed_data)
    # "morp" 키에서 "type"이 "NNG"이면서 "lemma"이 "학과"나 "과"가 아닌 행들만 추출
    filtered_morp = [item['lemma'] for item in parsed_data['return_object']['sentence'][0]['morp'] if item['type'] == 'NNG' and item['lemma'] not in ['학과', '공학', '계열', '분야', '전공', '학부']]
    # 제거: 1글자인 항목-> 정확도
    filtered_morp = [item for item in filtered_morp if len(item) > 1]
    # 중복 요소 삭제
    filtered_morp = list(set(filtered_morp))
    # print('필터몹: ',filtered_morp)
    new_df = pd.DataFrame()
    df = pd.read_csv('major_uni.csv')
    df = df.dropna(subset=['학부_과(전공)명'])

    all_results = ''
    next_string = '계속하시려면 돌아가서 "계속하기" 버튼을 꼭 눌러주세요.' 
    for word in filtered_morp:
        filtered_rows = df[df['학부_과(전공)명'].str.contains(word)]
        all_results += f'<{word}>분야와 관련하여 총 {len(filtered_rows)}개의 학과와 대학이 검색되었습니다.'
        if word != filtered_morp[-1]:
            all_results += '\n\n'
        new_df = pd.concat([new_df, filtered_rows])
        new_df = new_df.drop_duplicates(subset=['학부_과(전공)명', '학교명'])

    new_df.to_csv('new_df.csv', index=False, encoding='utf-8-sig')

    response = {
        "version": "2.0",
        "template": {
            "outputs": [
                {"simpleText": {"text": all_results}},
                {"simpleText": {"text": next_string}}
            ]
        }
    }

    return jsonify(response)


def chat_info_setting(text):
    global api_key, headers, data

    api_key = ""

    headers = {
        "Content-Type": "application/json",
        "Authorization": f"Bearer {api_key}"
    }

    data = {
        "model": "gpt-3.5-turbo",
        "messages": [
            {"role": "system", "content": "너는 진로상담사야."},
            {"role": "user", "content": text},
        ],
        "max_tokens": None,
        "n": 1,
        "temperature": 1,
    }

@application.route("/department_1", methods=["POST"])
def uni_department():
    global a
    global generated_text
    new_df = pd.read_csv('new_df.csv')
    request_data = json.loads(request.get_data(), encoding='utf-8')
    school_name = request_data['action']['params']['대학명']
    first_string = ""
    
    if school_name in new_df['학교명'].values:
        majors_related_to_uni = new_df['학부_과(전공)명'][new_df['학교명'].str.contains(school_name)].to_list()
        num_majors = len(majors_related_to_uni)
        first_string = "해당 분야와 관련하여 < {} > 에는 약 {} 개의 학과가 존재합니다.".format(school_name, num_majors)
        try:
            majors_df = pd.read_csv('cau_df.csv', usecols=['학과(부)/전공/영역'], encoding='cp949')
            majors_df.drop_duplicates(inplace=True)     
            filtered_majors = []
            for morp in majors_related_to_uni:
                filtered_majors_df = majors_df[majors_df['학과(부)/전공/영역'].apply(lambda x: fuzz.partial_ratio(x, morp) > 70)]
                filtered_majors += filtered_majors_df['학과(부)/전공/영역'].tolist()
            filtered_majors = list(set(filtered_majors))
            result_string = ", ".join(filtered_majors)
            next_string = '학과별 커리큘럼을 확인하시려면, "전공" 을 입력해주세요.'
            
        except Exception as e:
            print("Exception:", e)  # 예외 정보 출력
            first_string = ""
            error_string = '현재 해당 대학의 정보를 불러올 수 없습니다.\n다른 대학을 선택하시려면 "다시" 를 입력해주세요.'
            error_response = {
                "version": "2.0",
                "template": {
                    "outputs": [
                        {"simpleText": {"text": error_string}}
                    ]
                }
            }
            return jsonify(error_response)
    else:
        result_string = f'< {school_name} > 에는 해당 분야 관련 전공이 개설되어 있지 않습니다. 정확한 대학명을 입력해주세요.\n\n다른 대학을 선택하시려면 "다시" 를 입력해주세요.'
        next_string = ''  
    

    response = {
        "version": "2.0",
        "template": {
            "outputs": [
                {"simpleText": {"text": first_string}},
                {"simpleText": {"text": result_string}},
                {"simpleText": {"text": next_string}}
            ]
        } 
    }
    return jsonify(response)

@application.route("/department_3", methods=["POST"])
def export_cau():
    global a
    global generated_text
    global majors_related_to_uni
    global table_html
    # print('export_cau')
    courses_df = pd.read_csv('cau_df.csv', encoding='cp949')
    request_data = json.loads(request.get_data(), encoding='utf-8')
    # a[request_data['userRequest']['user']['id']] = '아직 AI가 처리중이에요'
    answer = request_data['action']['params']['커리큘럼']
    major_choice = answer
    # print(major_choice)

    if pd.notna(major_choice):
        if major_choice in courses_df['학과(부)/전공/영역'].values:
            courses_selected_df = courses_df.loc[courses_df['학과(부)/전공/영역'].str.contains(major_choice), ['학기', '학년', '이수구분', '과목명']]
            courses_selected_df = courses_selected_df.drop_duplicates(subset='과목명', keep='first')
            # Convert the DataFrame to a table in HTML format
            table_html = courses_selected_df.to_html(index=False)
            # print(table_html)
            # Create the response with the DataFrame table as HTML
            response = {
              "version": "2.0",
              "template": {
                "outputs": [
                  {
                    "textCard": {
                      "text": "자세한 커리큘럼을 확인하시려면 아래 버튼을 눌러주세요.",
                      "buttons": [
                        {
                          "action": "webLink",
                          "label": "커리큘럼 확인하러 가기",
                          "webLinkUrl": "http://cau-hk-zhpjc.run.goorm.site/send_table"
                        }
                      ]
                    }
                  }
                ]
              }
            }
            return jsonify(response)
        else:
            response = {
                "version": "2.0",
                "template": {
                    "outputs": [
                        {
                            "simpleText": {"text": '해당 대학에는 현재 {}가 존재하지 않습니다. 혹은 변동사항이 있을 수 있으니 해당 학교 사이트를 방문해보시길 바랍니다.\n\n다시 전공을 선택하시려면 "전공" 을 입력해주세요.'.format(major_choice)}
                        }
                    ]
                }
            }
        return jsonify(response)

@application.route("/send_table", methods=["GET"])
def send_table():
    global table_html
    return render_template("table_template.html", css=url_for('static', filename='style.css'), table_content=table_html)

if __name__ == "__main__":
    application.run(host='0.0.0.0', port=80, debug=True)
