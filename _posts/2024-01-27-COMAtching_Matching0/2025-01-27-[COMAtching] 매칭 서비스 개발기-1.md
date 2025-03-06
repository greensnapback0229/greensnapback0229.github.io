---
layout: post
title: "[COMAtching] 매칭 서비스 개발기-1"
date: 2025-01-01 13:35
category: [COMAtching, TIL]
author: [greensnapback0229]
tags: []
summary: 코매칭 서비스 개발기
---


> 코매칭 프로젝트를 하면서 매칭 기능을 담당해 계속 고도화 시킨 경험을 남기고자 합니다.
Ver1부터 3까지 고민했던 비즈니스 로직과 기술적 고민들에 관한 내용입니다.


## Ver1 매칭
Ver1에서는 정말 단순하게 crud 밖에 할줄 모르던 시기였고 그냥 이렇게 귀엽게 개발했었다는 내용입니다.

### Ver1 매칭 로직
Ver1은 MBTI 기반의 매칭이었습니다. 원하는 상대의 MBTI 중 E/I와 P/J를 골라서 해당하는 MBTI를 가진 유저를 랜덤으로 매칭해주었습니다.

```java
if (ei.equals("Z") && jp.equals("X")) {
			candidate = userInfoRepository.findByGenderAndChooseGreaterThan(gender, 0);
		} else if (jp.equals("X")) {
			candidate = userInfoRepository.findByMbtiStartingWithAndGenderAndChooseGreaterThan(ei, gender, 0);
		} else if (ei.equals("Z")) {
			candidate = userInfoRepository.findByMbtiEndingWithAndGenderAndChooseGreaterThan(jp, gender, 0);
		} else {
			candidate = userInfoRepository.findByMbtiStartingWithAndMbtiEndingWithAndGenderAndChooseGreaterThan(ei, jp,
				gender, 0);
		}
```
DB 설계도 부족했고 ORM을 무턱대고 쓰다보니 이렇게 되어버렸다.. 

## Ver2 매칭
Ver2부터 MBTI 외에도 다른 옵션들을 추가하는 기획이 있었고 나이, 연락빈도, MBTI, 취미와 같은 옵션을 추가하여 매칭을 하는 기능이 요구되었습니다.  
학교 축제 특성상 부스에서 행사가 진행되었고 운영자가 사용자의 QR을 인식하여 운영자의 테블릿에서 매칭을 옵션을 선택할 수 있는 참여형 
아래는 매칭 서비스 플로우입니다. 
![ver2_matching](/assets/comatching_2_matching_sequence_diagram.svg)


### AI 도입 
옵션들을 고려하며 정적으로 분기처리 하는 것은 너무나도 많은 경우의 수가 존재하기에 AI를 도입하여 존재하는 데이터중 최선의(best effort) 매칭 결과를 판단해주는 기능을 만들고자 했습니다. 

🧐 하지만 AI를 도입하면서 다음과 같은 요구사항이 있었습니다.
 - AI는 DB가 아닌 **CSV**를 참조하여 동작합니다. 즉, CSV와 DB의 동기화가 필요합니다.
 - 매칭시 AI를 모델을 실행시키고 결과를 읽어와야 합니다. 

이를 위해서 OpenCSV 라이브러리를 도입해 CSV 파일 관리를 위한 컴포넌트를 개발했습니다.
CSV파일을 접근할때 동시성 문제가 있었습니다. rrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrr

**\<example\>**  
사용자가 뽑힐 기회가 없어지면 CSV에서 제외되어야 하는데 쓰기 작업중에 파일을 읽게되면 해당 사용자가 포함된 상태로 매칭로직이 진행이됩니다.

이를 위해서 CSV 관리 컴포넌트에 Lock/Unlock 방식을 적용해서 동시에 여러 쓰레드가 파일에 접근하지 못하도록하여 문제를 해결했습니다. 
- CSV 접근하는 메서드가 호출될때 마다 `lock`을 걸어서 관련 처리를 완료하고 `finally`문을 활용하여 메서드가 모든 처리를 끝내면 `unlock`으로 해제해줍니다.
- 만약 lock을 획득하지 못하면 대기 상태로 들어가 현재 lock을 획득한 메서드의 종료를 기다리는 대기 상태(블로킹)으로 들어갑니다.

```java

@Component
public class CSVHandler {
	.
	.
	.
	
	private ReentrantLock lock = new ReentrantLock();

	/**
	 * CSV 유저 추가
	 * @param userInfo : 추가할 유저 정보
	 */
	public void addUser(UserInfo userInfo) {
		lock.lock();
		try {
			ICSVWriter csvWriter = new CSVWriterBuilder(new FileWriter(path, true))
				.withQuoteChar(ICSVWriter.NO_QUOTE_CHARACTER)
				.build();
			String[] newData = userAiFeatureToStringArray(userInfo, 1);
			csvWriter.writeNext(newData);
			csvWriter.close();
		} catch (IOException e) {
			throw new BusinessException(ResponseCode.MATCH_GENERAL_FAIL);
		} finally {
			lock.unlock();	//메서드 종료시 unlock
		}
	}

	/**
	 * CSV 매칭 정보 설정
	 * @param matchReq : 매칭 요청 정보
	 * @param pickerUsername : 매칭 요청한 유저의 username
	 */
	public void match(MatchReq matchReq, String pickerUsername) {
		lock.lock(); //메서드 호출시 lock

		try {
			CSVReader csvReader = new CSVReader(new FileReader(path));
			List<String[]> csvData = csvReader.readAll();
			csvReader.close();

			for (int i = 0; i < csvData.size(); i++) {
				String[] row = csvData.get(i);
				if (row[0].equals(pickerUsername)) {
					String[] s = setMatchOption(row, matchReq);
					csvData.set(i, setMatchOption(row, matchReq));
					break;
				}
			}

			ICSVWriter csvWriter = new CSVWriterBuilder(new FileWriter(path))
				.withQuoteChar(ICSVWriter.NO_QUOTE_CHARACTER)
				.build();

			csvWriter.writeAll(csvData);
			csvWriter.close();
		} catch (IOException | CsvException e) {
			throw new BusinessException(ResponseCode.MATCH_GENERAL_FAIL);
		} finally {
			lock.unlock();
		}
	}

		.
		.
		.
}
````

