# WEEK 14 Exercise

ให้นำแอปนัดหมายกับหมอในสัปดาห์ที่ 13 มาเพิ่มการทำ Authentication และ Authorization กันต่อ

โดยแต่ละ API จะมี endpoint และ ความต้องการทางด้าน Authentication และ Authorization ดังนี้

## API endpoints

    - GET /api/doctors/
        - จะต้อง `IsAuthenticated`

    - GET /api/patients/
        - จะต้อง `IsAuthenticated`

    - GET /api/appointments/
        - ผู้ใช้งานทั่วไป (ไม่ต้อง authenticate) สามารถเข้าถึงได้

    - GET /api/appointments/<id>/
        - จะต้อง `IsAuthenticated` และ มี permission `appointments.view_appointment`

    - POST /api/appointments/
        - จะต้อง `IsAuthenticated` และ มี permission `appointments.add_appointment`

    - PUT,PATCH /api/appointments/<id>/
        - จะต้อง `IsAuthenticated` และ มี permission `appointments.change_appointment`
        - จะต้องเป็นผู้ที่สร้างนัดหมายเท่านั้น (created_by)

    - DELETE /api/appointments/<id>/
        - จะต้อง `IsAuthenticated` และ มี permission `appointments.delete_appointment`
        - จะต้องเป็นผู้ที่สร้างนัดหมายเท่านั้น (created_by)

ก่อนอื่นเราจะต้องมาปรับแก้ไข model Appointment ใน `models.py` กันก่อน โดยเพิ่ม field `created_by` ซึ่ง FK ไปหา `User`

```python
...
class Appointment(models.Model):
    doctor = models.ForeignKey(Doctor, on_delete=models.CASCADE)
    patient = models.ForeignKey(Patient, on_delete=models.CASCADE)
    date = models.DateField()
    at_time = models.TimeField()
    details = models.TextField(null=True, blank=True)
    created_by = models.ForeignKey('auth.User', null=True, on_delete=models.PROTECTED)

    def __str__(self):
        return f'Appointment with Dr. {self.doctor.name} for {self.patient.name} on {self.date.strftime("YYYY-MM-DD HH:mm")}'
```

## Exercises

1. ให้ทำการติดตั้ง และใช้งาน `TokenAuthentication` ของ REST framework (0.5 คะแนน)

**Hint**: อย่าลืมไปทำการสร้าง user ในหน้า Django admin page

2. เพิ่ม endpoint สำหรับ generate token (0.5 คะแนน)

**Hint**: `path("api-token-auth/", views.obtain_auth_token)`

3. เพิ่มการตรวจสอบ permission - `IsAuthenticated` และ permission ของ model `Appointment` (0.5 คะแนน)

4. เพิ่มการตรวจสอบ กรณี PUT/DELETE จะต้องตรวจสอบด้วยว่าผู้ทำเป็นเจ้าของ Appointment นั้น `(request.user == appointment.created_by)` (0.5 คะแนน)
