using Rocket.API;
using Rocket.Unturned.Chat;
using Rocket.Unturned.Player;
using Rocket.Unturned.Events;
using SDG.Unturned;
using UnityEngine;
using System.Collections;
using System.Collections.Generic;

public class KnockdownPlugin : RocketPlugin<KnockdownPluginConfiguration>
{
    private Dictionary<UnturnedPlayer, Coroutine> knockdownPlayers = new Dictionary<UnturnedPlayer, Coroutine>();

    protected override void Load()
    {
        // Подписываемся на событие изменения здоровья игрока
        UnturnedPlayerEvents.OnPlayerUpdateHealth += OnPlayerUpdateHealth;
        // Подписываемся на событие лечения игрока
        UnturnedPlayerEvents.OnPlayerHealed += OnPlayerHealed;
        UnturnedChat.Say("Плагин KnockdownPlugin загружен!", Color.green);
    }

    protected override void Unload()
    {
        // Отписываемся от событий при выгрузке плагина
        UnturnedPlayerEvents.OnPlayerUpdateHealth -= OnPlayerUpdateHealth;
        UnturnedPlayerEvents.OnPlayerHealed -= OnPlayerHealed;
        UnturnedChat.Say("Плагин KnockdownPlugin выгружен!", Color.red);
    }

    private void OnPlayerUpdateHealth(UnturnedPlayer player, byte health)
    {
        // Проверяем, что здоровье игрока упало до 10% или ниже
        if (health <= player.MaxHealth * 0.1f && !knockdownPlayers.ContainsKey(player))
        {
            // Выводим сообщение игроку
            UnturnedChat.Say(player, "Вы потеряли сознание! Вам нужна помощь.", Color.red);

            // Запускаем корутину для обработки состояния "нока"
            Coroutine knockdownCoroutine = StartCoroutine(KnockdownPlayer(player));
            knockdownPlayers.Add(player, knockdownCoroutine);
        }
    }

    private void OnPlayerHealed(UnturnedPlayer healer, UnturnedPlayer target, byte amount)
    {
        // Если игрок, которого лечат, находится в состоянии нока
        if (knockdownPlayers.ContainsKey(target))
        {
            // Останавливаем корутину нока
            StopCoroutine(knockdownPlayers[target]);
            knockdownPlayers.Remove(target);

            // Восстанавливаем возможность двигаться
            target.Player.movement.sendPluginSpeedMultiplier(1f);
            target.Player.movement.sendPluginGravityMultiplier(1f);
            target.Player.movement.sendPluginJumpMultiplier(1f);

            // Отключаем анимацию ползания
            target.Player.animator.setPluginBool("Crawling", false);

            // Сообщаем игрокам
            UnturnedChat.Say(healer, $"Вы спасли {target.DisplayName}!", Color.green);
            UnturnedChat.Say(target, $"{healer.DisplayName} спас вас! Вы можете двигаться.", Color.green);
        }
    }

    private IEnumerator KnockdownPlayer(UnturnedPlayer player)
    {
        // Отключаем возможность двигаться и стоять
        player.Player.movement.sendPluginSpeedMultiplier(0f); // Останавливаем движение
        player.Player.movement.sendPluginGravityMultiplier(0f); // Отключаем гравитацию
        player.Player.movement.sendPluginJumpMultiplier(0f); // Отключаем прыжки

        // Включаем анимацию ползания
        player.Player.animator.sendPluginAnimSpeed(0.5f); // Замедляем анимацию
        player.Player.animator.setPluginBool("Crawling", true); // Включаем ползание

        // Ждём указанное время до смерти
        yield return new WaitForSeconds(Configuration.Instance.KnockdownDuration);

        // Если игрок не был спасён, он умирает
        if (knockdownPlayers.ContainsKey(player))
        {
            player.Damage(101, EDamageOrigin.Player); // Убиваем игрока
            UnturnedChat.Say(player, "Вам не успели помочь...", Color.red);
            knockdownPlayers.Remove(player);
        }
    }
}

public class KnockdownPluginConfiguration : IRocketPluginConfiguration
{
    public float KnockdownDuration { get; set; } // Длительность нока (в секундах)

    public void LoadDefaults()
    {
        KnockdownDuration = 300f; // 5 минут по умолчанию
    }
}
