using System;
using System.Collections.Generic;
using System.Linq;
using cowsins;
using UnityEngine;
using UnityEngine.Events;

public class EXPUIController : MonoBehaviour
{
    public List<ChooseUI> UI;
    public List<UpgradeData> SOList;
    private List<TempData> temps = new List<TempData>();

    public cowsins.PlayerMovement Movement;
    public WeaponController Weapon;

    public static bool isPaused { get; private set; }
    public static EXPUIController Instance { get; private set; }
    
    public PlayerStats stats;

    [SerializeField] private float fadeSpeed;
    [SerializeField] private bool disablePlayerUIWhilePaused;
    [SerializeField] private CanvasGroup menu;
    [SerializeField] private GameObject playerUI;


    private void Awake()
    {
        if (Instance != null && Instance != this) Destroy(this);
        else Instance = this;

        isPaused = false;
        menu.gameObject.SetActive(false);
        menu.alpha = 0;

        for (int i = 0; i < SOList.Count; i++)
        {
            TempData data = new TempData();
            data.SetType(SOList[i].Type);

            temps.Add(data);
        }
        Debug.Log(JsonUtility.ToJson(temps));
    }

    private void Start() => ExperienceManager.instance.OnLevelUp += TogglePause;

    private void Update()
    {
        if (isPaused)
        {
            stats.LoseControl();
            if (!menu.gameObject.activeSelf)
            {
                menu.gameObject.SetActive(true);
                menu.alpha = 0;
            }
            if (menu.alpha < 1) menu.alpha += Time.deltaTime * fadeSpeed;

            if (disablePlayerUIWhilePaused && !stats.isDead) playerUI.SetActive(false);
        }
        else
        {
            menu.alpha -= Time.deltaTime * fadeSpeed;
            if (menu.alpha <= 0) menu.gameObject.SetActive(false);
        }
    }

    public void TogglePause()
    {
        isPaused = !isPaused;
        if (isPaused)
        {
            Cursor.lockState = CursorLockMode.None;
            Cursor.visible = true;

            if (disablePlayerUIWhilePaused && !stats.isDead) playerUI.SetActive(false);
            SetChooseCard();

            Time.timeScale = 0.1f;
        }
        else
        {
            UnPause();
        }
    }

    public void UnPause()
    {
        isPaused = false;
        stats.CheckIfCanGrantControl();
        Cursor.lockState = CursorLockMode.Locked;
        Cursor.visible = false;
        Time.timeScale = 1f;

        playerUI.SetActive(true);
    }

    public void SetChooseCard() 
    {
        for (int i = 0; i < UI.Count; i++)
        {
            var so = GetRandomUpgrade();
            UI[i].Construct(so);
        }
    }

    public UpgradeData GetRandomUpgrade()
    {
        var availableSO = SOList.Where(so => GetUsageCount(so) < so.MaxUsageCount).ToList();

        if (availableSO.Count == 0)
            return null;
        
        var randomSO = availableSO.OrderBy(so => Guid.NewGuid()).First();

        return randomSO;
    }

    public void HandleUpgradeData(UpgradeData data)
    {
        var currentTemp = temps.Find(t => t.Type == data.Type);
        currentTemp.Counter++;

        switch (data.Type)
        {
            case UpgradeType.Speed:

                var current = Movement.walkSpeed;
                float newValue = current * (1f + data.Modificator / 100f);
                Movement.walkSpeed = newValue;

                var currentRun = Movement.runSpeed;
                float newValueRun = currentRun * (1f + data.Modificator / 100f);
                Movement.runSpeed = newValueRun;

                break;
            case UpgradeType.Attack:
                break;
            case UpgradeType.Jump:
                Movement.maxJumps++;
                break;
            case UpgradeType.Dash:
                Movement.amountOfDashes++;
                break;
            case UpgradeType.Weapon_Speed:
                var currentFR = Weapon.inventory[0].weapon.fireRate;
                float newFR = currentFR * (1f + data.Modificator / 100f);
                Weapon.inventory[0].weapon.fireRate = newFR;
                break;
            case UpgradeType.Weapon_Attack:

                var currentAttack = Weapon.inventory[0].weapon.damagePerBullet;
                float newAttack = currentAttack * (1f + data.Modificator / 100f);
                Weapon.inventory[0].weapon.damagePerBullet = newAttack;
                Weapon.damagePerBullet = newAttack;

                break;
        }

        TogglePause();
    }


    private int GetUsageCount(UpgradeData so)
    {
        var currentTemp = temps.Find(t => t.Type == so.Type);
        return currentTemp.Counter;
    }
}

public enum UpgradeType 
{
    Speed = 0,
    Attack = 1,
    Jump = 2,
    Dash = 3,
    Weapon_Speed = 4,
    Weapon_Attack = 5
}
