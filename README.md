# meetingroom


# 서비스 시나리오

1. 비품 관리를 위한 서비스를 미리 생성한다.
2. 회의실 등록을 하면 비품이 등록된다.
3. 관리자에 의해 등록된 비품은 수정될 수 있다.
4. 회의실을 제거하면 비품 항목도 모두 삭제한다.


비기능적 요구사항
1. 트랜잭션
  - 회의실 삭제 시 비품서비스가 없다면 삭제하지 못하게 한다. 생성단계부터 오류가 있는 것으로 확인 필요 (Sync 호출) 
2. 장애격리
  - 비품 서비스가 수행되지 않더라도 회의실 생성은 365일 24시간 생성되어야 회의 진행하는데 문제가 없다. (Async 호출)
3. 성능
  - 비품 현황에 대해 별도록 확인할 수 있어야 한다. (CQRS)


# 체크포인트
https://workflowy.com/s/assessment/qJn45fBdVZn4atl3


# 모형
![모형](https://user-images.githubusercontent.com/78134049/109769512-a8191f00-7c3d-11eb-88bb-334660ee98be.png)

# 기능적/비기능적 요구사항에 대한 검증
![모형2](https://user-images.githubusercontent.com/78134049/109770245-ad2a9e00-7c3e-11eb-9b18-1091ffd17ee0.png)

1. 비품 등록 서비스를 생성한다. (1)
2. 관리자가 회의실을 생성 및 등록한다.(2)
   - 생성하면 비품이 자동 등록된다. (2 -> 4)
3. 관리자가 회의실을 삭제 한다.(3) 
   - 삭제하면 비품 항목을 전부 삭제한다. (3 -> 5)
4. Stock 메뉴에서 회의실에 대한 예약 정보를 조회한다.(6)

# 헥사고날 아키텍쳐 다이어그램 도출 (Polyglot)

# 구현

# DDD의 적용

package meetingroom;

import javax.persistence.*;
import org.springframework.beans.BeanUtils;
import java.util.List;

@Entity
@Table(name="Supplies_table")
public class Supplies {
    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long id; 
    private int phone;
    private int pc;
    private int beam;
    
    @PrePersist
    public void onPrePersist(){
    }
    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public int getPhone() {
        return phone;
    }

    public void setPhone(int phone) {
        this.phone = phone;
    }
    public int getPc() {
        return pc;
    }

    public void setPc(int pc) {
        this.pc = pc;
    }
    public int getBeam() {
        return beam;
    }

    public void setBeam(int beam) {
        this.beam = beam;
    }
}
