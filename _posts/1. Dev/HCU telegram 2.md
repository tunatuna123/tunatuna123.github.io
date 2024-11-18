---
title: "HCU telegram 2편"
excerpt: "청운네트워크 급식 신청"

categories:
  - Dev
  - HCU telegrambot
tags:
  - [Python, 1인 개발]

permalink: /categories1/HCU telegram 2/

toc: true
toc_sticky: true

date: 2024-02-12
last_modified_at: 2024-02-12
---

# 청운네트워크 급식 신청

사실 HCU telegrambot을 만들어야겠다고 결정적인 다짐을 하게 된 이유는 매주 주말 급식 신청을 잊어버려 '자동으로 좀 급식을 신청할 수 없을까?'라는 생각을 하면서이다. 그렇기에 가장 먼저 개발을 시작한 것이 급식 자동 신청과 관련한 부분이다. 이번 편에서는 내가 어떻게 급식 신청 알고리즘을 만들었는지, 그리고 급식 신청 자동화를 개발하며 어떤 어려움이 있었는지를 기록해 보겠다.

## 청운네트워크 급식 신청 과정

급식 신청을 자동화 하겠다고 했으면 먼저 청운네트워크가 어떻게 급식을 신청하는지를 알아야했다. 일단 먼저 내가 짜놓은 급식 신청 로직 코드를 보여주겠다. 참고로 아래 코드는 자동 신청이 아니라 수동 신청에 관한 코드이다. 어쨋건 과정을 보여주기에는 해당 코드가 좋으니 이를 사용해겠다.

```python
# 주말 급식 신청
if text == '주말 급식 신청':
    user_id = update.message.from_user['id']
    html = scrape_hcu('https://hcuhs.kr/meal/student.php', user_id)
    soup = BeautifulSoup(html, 'html.parser')
    
    input_tag_list = soup.find_all('input', {'class':'cbx-tea2', 'id':True})
    button_name_list = []
    for i in input_tag_list:
        button_name_list.append(i['name'])
    button_name_list.append('')
    
    date_tag_list = soup.find_all('td', {'style':'border-width:1;border-color:gray;border-style:solid'})
    date_tag_list = list(map((lambda x: list(x.stripped_strings)[0]),(date_tag_list)))
    meal_menu_date_list = ['/'.join(date_tag_list[i * 3:(i + 1) * 3])for i in range((len(date_tag_list) + 3 - 1) // 3 )][1:]
    meal_menu_date_list.append('모두 취소')

    date_js_name_dict = {name: value for name, value in zip(meal_menu_date_list, button_name_list)}
    message = await context.bot.send_poll(
        update.effective_chat.id,
        "급식을 신청할 날짜를 선택하세요",
        meal_menu_date_list,
        is_anonymous=False,
        allows_multiple_answers=True,
        reply_markup=markup
    )
    
    payload = {
        message.poll.id: {
            "questions": meal_menu_date_list,
            "chat_id": update.effective_chat.id,
            "date_js_name_dict": date_js_name_dict,
        }
    }
    context.bot_data.update(payload)
```

먼저 `scrape_hcu()`는 `url`과 `user_id`를 인자로 가지는데, 이는 1편에 언급한 telegram의 고유 유저 아이디인 `user_id`를 이용해 로그인을 하고, `url`에 해당하는 사이트로 접속하는 함수이다. 1편에서 보여준 코드를 조금 더 일반화 시킨 것인데, 아래와 같다.

```python
# scrape hcu by login
def scrape_hcu(url, id) -> str:
    while True:
        conn,cur = connect_RDS(host, port, username, password, database)
        cur.execute("SELECT student_num FROM user_info WHERE id=(%(user_id)s)", {'user_id': str(id)})
        student_num = cur.fetchall()[0][0]
        cur.execute("SELECT pw FROM user_info WHERE id=(%(user_id)s)", {'user_id': str(id)})
        pw = cur.fetchall()[0][0]
        conn.commit()
        conn.close()

        with requests.Session() as s:
            data = {
                'sel': 'stu',
                'text1': student_num,
                'password': pw,
            }
            s.post('https://hcuhs.kr/login_check.php', headers=headers, data=data)
            response = s.get(url=url, headers=headers)
            response.encoding = 'EUC-KR'
            if response.status_code == 200:
                if '접속이 끊어졌습니다' not in response.text:
                    return response.text
```

`scrape_hcu()`를 통해 급식 신청 사이트의 정보를 가져온 이후에는 `BeautifulSoup`을 이용해서 모든 급식 신청 버튼을 찾아주었다. 이후 `lambda`와 `map`을 이용해 주말 급식 신청 버튼만을 찾고, 날짜와 급식 신청 버튼의 요소값을 dictionary로 만들어줬다. 그리고 `telegram` 모듈을 이용하여 이를 contextbox로 유저의 화면에 띄우게 했다. 해당 영상이 남아있는지 확인하고 싶었지만 아쉽게도 급식 신청과 관련해 남아있는 영상 자료는 없었고, 